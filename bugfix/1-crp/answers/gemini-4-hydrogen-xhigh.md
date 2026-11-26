# Stage 1 静态分析与修复报告（CoralReefPlayer RTP 组播画面撕裂/马赛克问题）

## 1. 现象与约束分析

1. **故障现象**（结合 `image1.png`、`image2.png`）：
   - 直连同一 IPC 设备（无网络丢包、Wireshark 抓包完整且 VLC 播放 RTSP 与 RTP 均正常），播放器通过 RTSP 播放正常，通过 RTP 组播在画面变化时出现严重马赛克、下半部分纵向拉丝（`image1.png`）及运动物体跨边界撕裂/重影抖动（`image2.png` 红框中的手部）。
   - 该画面特征属于 **H.264/H.265 解码器内部码流残缺或半帧错误隐藏（Error Concealment）** 引起的宏块纵向拉伸与参考帧局部错位，而非单缓冲区渲染竞态（已由题目与 `README.md` 排除）或 B 帧重排问题（IPC 码流无 B 帧）。

## 2. 静态根因定位

围绕 `StreamPuller::runRTP()`、`thirdparty/live555`、`StreamSink` 与 `VideoDecoder` 的完整代码路径排查，定位到三个导致 RTP 组播在画面变化时发生码流截断/丢包/半帧解码的核心缺陷：

### 根因一（核心：`PT=33` MPEG-2 TS over RTP 解复用路径）：`MPEG2TransportStreamDemuxedTrack::maxFrameSize()` 未实现（返回 `0`）导致 `StreamParser` 缓冲区回绕时截断并丢弃视频码流
- **代码路径**：
  - `src/StreamPuller.cpp`：`StreamPuller::runRTP()` 在 `payloadType == 33` 时创建 `MPEG2TransportStreamSource` -> `MPEG2TransportStreamDemux`，并在 `createTransportStream()` 中将 `pidState->streamSource`（`MPEG2TransportStreamDemuxedTrack*`）传给 `H264VideoStreamFramer` / `H265VideoStreamFramer`。
  - `thirdparty/live555/liveMedia/FramedSource.cpp`（`CRP_MODIFY=1`）中 `FramedSource::maxFrameSize()` 返回 `fMaxFrameSize`（默认 `0`）。
  - `thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh/.cpp` 未重写 `maxFrameSize()`，因此返回 `0`。
  - `H264or5VideoStreamParser` 继承自 `StreamParser`（`thirdparty/live555/liveMedia/StreamParser.cpp`，双 bank，每 bank 大小 `BANK_SIZE = 150000`）。在 `StreamParser::ensureValidBytes1(numBytesNeeded)` 中：
    ```cpp
    unsigned maxInputFrameSize = fInputSource->maxFrameSize(); // 为 0
    if (maxInputFrameSize > numBytesNeeded) numBytesNeeded = maxInputFrameSize;
    if (fCurParserIndex + numBytesNeeded > BANK_SIZE) { /* swap banks */ }
    unsigned maxNumBytesToRead = BANK_SIZE - fTotNumValidBytes;
    fInputSource->getNextFrame(&curBank()[fTotNumValidBytes], maxNumBytesToRead, ...);
    ```
  - 当 `BANK_SIZE - fTotNumValidBytes` 小于一个 TS 负载长度（最多 184 字节）且 `fCurParserIndex + 4 <= BANK_SIZE` 时，`StreamParser` **不会切换 bank**，而是以 `maxSize = 1..183` 请求下一个 TS 包负载。
  - 随后在 `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp` `processStreamPacket()` 中：
    ```cpp
    if (numBytesToDeliver > streamSource->maxSize()) {
      streamSource->frameSize() = streamSource->maxSize();
      streamSource->numTruncatedBytes() = numBytesToDeliver - streamSource->maxSize();
    }
    getBytes(streamSource->to(), streamSource->frameSize());
    skipBytes(streamSource->numTruncatedBytes());
    ```
    **直接跳过并永久丢弃了剩余的 1~183 字节视频 ES 数据**！
- **与现象对应**：
  - 静态画面码率低、触发 150 KB 回绕少且静止宏块错误隐藏不易察觉；**画面变化时码率陡增**，150 KB bank 频繁回绕并在 slice 中间丢失字节，导致熵解码失步、下半帧错误隐藏拉丝（`image1.png`）与运动手部参考帧错位撕裂（`image2.png`）。
  - Wireshark 抓包位于网络层，无丢包；VLC 使用自身 TS 解复用器，不受 live555 `StreamParser` 影响；RTSP 走 `H264VideoRTPSource`（无 TS 与 `H264VideoStreamFramer`），故 RTSP 播放正常。
- **关联隐患**：
  - `src/StreamPuller.cpp` 中 `MAX_TS_FRAME_SIZE` 仅为 `7 * TRANSPORT_PACKET_SIZE`（1316 字节），若单个 UDP/RTP 包携带超过 7 个 TS 包也会被 `BufferedPacket::use()` 截断（且 `StreamParser::afterGettingBytes` 忽略 `numTruncatedBytes`）。
  - `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp` 中 `PIDState_STREAM` 在 `CRP_MODIFY=1` 时未初始化 `streamSink` 指针，析构时会释放野指针；且 `StreamPuller::createTransportStream()` 遇到不支持的 PID（如音频/私有数据）未关闭并置空 `pidState->streamSource`，会导致 `MPEG2TransportStreamParser` 因 `!streamSource->isCurrentlyAwaitingData()` 永久停滞。

### 根因二（RTP 传输层缓冲与突发阻塞）：`MultiFramedRTPSource` UDP 接收缓冲区过小（50 KB）且解码阻塞事件循环线程
- **代码路径**：
  - `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp` 构造函数仅调用 `increaseReceiveBufferTo(env, RTPgs->socketNum(), 50*1024)`（`MediaSession.cpp` 最小也仅 `50*1024`、`BasicUDPSource.cpp` 同样 `50*1024`）。
  - 在 `StreamPuller::runRTP()` 中，`BasicTaskScheduler::SingleStep()` 每次 `select()` 仅读取 1 个 UDP 包，并在同一拉流线程内同步通过 `StreamSink::afterGettingFrame()` -> `videoCallback` -> `VideoDecoder::processPacket()` 执行软解（`avcodec_send_packet`/`avcodec_receive_frame`）与 `sws_scale()` 图像缩放/格式转换。
- **与现象对应**：
  - 在 Windows（默认 `SO_RCVBUF` 为 64 KB，`50*1024 < 65536` 导致 `increaseReceiveBufferTo` 直接跳过）或小缓冲环境下，当画面剧烈变化（关键帧或大运动 P 帧突发发送大量 UDP 分片包）且线程正在执行软解/`sws_scale` 时，内核 UDP 接收缓冲区溢出丢包，丢失帧尾部或中间分片，造成画面底部拉丝与局部撕裂；而 Wireshark 抓包在网卡驱动层捕获（无丢包），VLC 会调大 `SO_RCVBUF` 并异步解复用/解码，因此无此问题。

### 根因三（解码喂包粒度）：`AV_CODEC_FLAG2_CHUNKS` 被禁用而 `StreamSink` 按单个 NALU 送入 `avcodec_send_packet()`
- **代码路径**：
  - `src/VideoDecoder.cpp` 第 166 行注释掉了 `codecCtx->flags2 = AV_CODEC_FLAG2_CHUNKS; // BUG have bug when using drm on linux`。
  - `H264VideoRTPSource`（PT 96..127）与 `H264VideoStreamFramer`（PT 33 TS）每次 `getNextFrame()` 均交付**单个 NAL 单元**，而非完整视频帧（Access Unit）。
  - `src/StreamSink.cpp` 在 `afterGettingFrame()` 中每收到一个 NALU 就直接构造 `AVPacket` 送入 `VideoDecoder::processPacket()`。
- **与现象对应**：
  - 当画面变化导致单帧码流增大、编码器将一个帧切分为多个 Slice（multi-slice）时，FFmpeg 在未开启 `AV_CODEC_FLAG2_CHUNKS` 时把每个 Slice 当作完整图像结束帧解码：第 1 个 Slice 解码后立即对下半帧做错误隐藏（纵向拉丝，如 `image1.png`），第 2 个 Slice 又被当作新帧解码并对上半帧做错误隐藏（旧帧上半部 + 新帧下半部，导致运动物体撕裂与画面抖动，如 `image2.png`）。
  - 在 `StreamSink` 层按 Access Unit 边界将同一帧的所有 NALU（含 SPS/PPS/SEI 及多个 Slice）聚合为单个 `AVPacket` 后送入解码器，既可彻底解决多 Slice 撕裂与抖动，又无需开启 `AV_CODEC_FLAG2_CHUNKS`（保持与 Linux DRM 等硬解后端兼容，且单 Slice 与标准 RTP Marker 位码流零额外帧延迟）。

## 3. 修改清单与说明

### 第三方库修改（`thirdparty/live555`）
1. `thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh` & `.cpp`
   - 在 `#if CRP_MODIFY` 下重写 `MPEG2TransportStreamDemuxedTrack::maxFrameSize()` 返回 `512`（大于单 TS 包 188 字节），确保下游 `H264or5VideoStreamParser`（`StreamParser`）在剩余空间不足一个 TS 负载前先切换 bank，不再触发截断丢字节。
2. `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp`
   - 在 `PIDState_STREAM` 构造函数初始化列表（`#if CRP_MODIFY`）显式初始化 `streamSink(NULL)`，防止未创建 sink 时析构崩溃。
3. `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`、`MediaSession.cpp`、`BasicUDPSource.cpp`
   - 在 `#if CRP_MODIFY` 下将 UDP/RTP 接收缓冲区下限从 `50*1024` 提升至 `2*1024*1024`（2 MB，`increaseReceiveBufferTo` 会在系统上限不足时自动折半适配），避免大帧突发溢出丢包。
4. `thirdparty/live555/liveMedia/H264VideoRTPSource.cpp` & `H265VideoRTPSource.cpp`
   - 在 `#if CRP_MODIFY` 下为聚合包（STAP-A/B、MTAP16/24 与 HEVC AP）显式设置 `fCurrentPacketBeginsFrame = fCurrentPacketCompletesFrame = True`，避免前序分片丢包残留状态导致后续参数集聚合包被误弃。

### 播放器源码修改（`src`）
5. `src/StreamPuller.cpp`
   - 将 `MAX_TS_FRAME_SIZE` 调整为 `174 * TRANSPORT_PACKET_SIZE`（约 32 KB，远小于 `StreamParser::BANK_SIZE` 150000），避免单包超 7 个 TS 包被截断。
   - 在 `StreamPuller::runRTP()` 创建 `rtpSocket` 后立即调用 `increaseReceiveBufferTo(..., 2 * 1024 * 1024)`；使用 64 KB 缓冲区循环等待首个有效 RTP 包（`packetSize >= 12` 且 RTP version == 2）以探测 Payload Type，并将动态 Payload Type 支持范围修正为 `96..127`。
   - 在 `StreamPuller::createTransportStream()` 中遇到不支持的视频编码或非视频流 PID 时，调用 `Medium::close(pidState->streamSource); pidState->streamSource = NULL;` 释放并置空 `streamSource`，使 `MPEG2TransportStreamParser` 跳过该 PID 数据而不阻塞视频流解析。
6. `src/StreamSink.h` & `src/StreamSink.cpp`
   - 将 `SINK_RECEIVE_BUFFER_SIZE` 提升至 2 MB，并在传递给 FFmpeg 的缓冲区末尾保留并清零 `AV_INPUT_BUFFER_PADDING_SIZE` 填充字节；修复 `presentationTime.tv_sec * 1000` 在 Windows 32 位 `long` 下的乘法溢出（强转 `int64_t`）。
   - 为 H.264/H.265 新增 Access Unit（完整帧）聚合逻辑：
     - 结合 RTP Marker 位（`RTPSource::curPacketMarkerBit()`）或 TS 流 `H264VideoStreamFramer`/`H265VideoStreamFramer` 的 `pictureEndMarker()`（在 `continuePlaying()` 预先清零）判断帧结束，并辅以新帧起始 NALU 检测（`AUD/SPS/PPS/VPS/PREFIX_SEI` 或 Slice 首宏块/首切片标志 `first_mb_in_slice == 0` / `first_slice_segment_in_pic_flag == 1` 或 VCL PTS 变化）作为兜底刷新条件；在 `onSourceClosure` 时冲刷未发送的完整 VCL 帧，在 `stopPlaying()` 时复位状态。
     - 单 Slice 且满足帧结束标记的常见帧走零拷贝直通路径，不引入额外拷贝与帧延迟。
7. `src/VideoDecoder.cpp`
   - 将 `EXTRADATA_MAX_SIZE` 提升至 `1024` 并在 `addExtraData()` 中清零尾部 `AV_INPUT_BUFFER_PADDING_SIZE` 字节；为 `extractH264Or5Frame()` 与 H.264/H.265 NALU 类型解析补齐起始码及头部长度边界检查。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Gemini 4 Hydrogen (XHigh)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 通过（16m38s）；12 文件（+260 / -42）；pcap1 `956 / 0 / 0`，pcap2 `845 / 0 / 0`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
