# CoralReefPlayer RTP 组播画面撕裂与马赛克问题静态分析与修复报告（阶段 1）

## 1. 问题现象与关键线索分析

结合题目描述与两张问题截图（`image1.png`、`image2.png`）：

- **截图 1（`image1.png`）**：画面上半约 60% 清晰完整，下半约 40% 在某一特定宏块行中间位置（左侧约至行中段，右侧已断在上一行）突然出现典型的 H.264/H.265 光栅扫描序解码中断与垂直拉伸涂抹（FFmpeg 帧内/空间错误隐藏特征）。这直接表明送入 FFmpeg 解码器的单帧码流在某一宏块处发生**字节截断、中间字节丢失，或仅收到完整图像的前部 Slice**。
- **截图 2（`image2.png`）**：除底部仍残留花屏/涂抹外，右上红框内快速移动的手部出现明显的错位“撕裂”与重影。在无 B 帧的 IPC 码流中，这源于前序帧部分宏块损坏/丢失或单帧多 Slice 被拆成多次 `avcodec_send_packet()` 解码：帧间预测（P 帧）运动矢量叠加在半旧半新的参考帧或被错误隐藏的宏块区域上，导致运动物体在不同宏块行呈现不同时刻的画面并伴随抖动。
- **关键对比条件**：
  1. 摄像头直连，Wireshark 抓包无丢包且抓包文件可正常播放 -> **链路层无丢包，数据丢失/破坏发生在操作系统 Socket 缓冲层或播放器应用层（live555 / StreamSink）**。
  2. 同一设备 RTSP 播放正常，仅 RTP 组播在**画面变化时**（码率突增、单帧体积显著变大）触发花屏与撕裂。
  3. VLC 播放 RTSP 与 RTP 均正常。
  4. 项目遵循单缓冲区低延迟设计且码流不含 B 帧 -> **不改变单缓冲区架构，不引入抗抖动帧缓冲队列**。

---

## 2. 根因分析

通过静态审计 `src/` 与 `thirdparty/live555/` 全部拉流、解复用、组帧与解码链路，定位到导致 RTP 组播在画面变化时出现字节丢失、内核丢包和半帧错误隐藏的核心根因如下：

### 根因一（核心）：MPEG-2 TS 二级 `StreamParser` 缺少 `maxFrameSize()` 且 `StreamParser` 换页判断存在写指针越界截断（`thirdparty/live555`）

在 `StreamPuller::runRTP()` 中，当 RTP 组播为 MPEG-2 TS（`payloadType == 33`）时，完整数据流为：

```
Groupsock -> MPEG2TransportStreamSource (SimpleRTPSource)
          -> MPEG2TransportStreamDemux / MPEG2TransportStreamParser (第 1 级 StreamParser)
          -> MPEG2TransportStreamDemuxedTrack
          -> H264VideoStreamFramer / H265VideoStreamFramer -> H264or5VideoStreamParser (第 2 级 StreamParser)
          -> StreamSink -> VideoDecoder
```

作者此前已在第 1 级 `StreamParser` 的输入源 `MPEG2TransportStreamSource`（以及 RTSP MP2T 的 `subsession->readSource()`）上添加了 `maxFrameSize()`（注释位于 `src/StreamPuller.cpp:64-70`），但**遗漏了第 2 级 `StreamParser`（`H264or5VideoStreamParser`）的输入源 `MPEG2TransportStreamDemuxedTrack`**：

1. **`MPEG2TransportStreamDemuxedTrack::maxFrameSize()` 返回 `0`**：
   - `MPEG2TransportStreamDemuxedTrack` 继承自 `FramedSource`，未覆写 `maxFrameSize()`，亦未调用 `setMaxFrameSize()`，因此 `fInputSource->maxFrameSize()` 恒返回 `0`。
   - `H264or5VideoStreamParser` 每次扫描起始码调用 `test4Bytes()` -> `ensureValidBytes(4)` -> `StreamParser::ensureValidBytes1(4)`（`thirdparty/live555/liveMedia/StreamParser.cpp:121-162`）。
   - `BANK_SIZE = 150000`，每个 TS 包有效载荷最高 `184` 字节（`150000 = 815 * 184 + 40`）。当 `fTotNumValidBytes` 接近 `150000`（如 `149960`，`fCurParserIndex = 149957`）时，由于 `maxInputFrameSize == 0`，`fCurParserIndex + 4 <= 150000` 成立，**不会触发 Bank 切换**，而是以 `maxNumBytesToRead = BANK_SIZE - fTotNumValidBytes`（仅 `40` 字节，小于 184 字节）调用 `MPEG2TransportStreamDemuxedTrack::getNextFrame()`。
   - 在 `MPEG2TransportStreamParser::processStreamPacket()`（`thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp:53-62`）中：
     ```cpp
     if (numBytesToDeliver > streamSource->maxSize()) {
       streamSource->frameSize() = streamSource->maxSize();
       streamSource->numTruncatedBytes() = numBytesToDeliver - streamSource->maxSize();
     }
     getBytes(streamSource->to(), streamSource->frameSize());
     skipBytes(streamSource->numTruncatedBytes()); // 永久丢弃截断字节！
     ```
     超出 `maxSize()` 的最多 `183` 字节 H.264/H.265 裸流被 `skipBytes()` **静默永久丢弃**，而 `StreamParser::afterGettingBytes()` 完全忽略 `numTruncatedBytes`，不报错也不崩溃。
   - **与现象完全吻合**：每消耗 150 KB 视频裸流必定静默切掉一截 H.264/H.265 字节；静止画面码率低、触发稀疏；**画面剧变（挥手）时码率飙升，每秒多次在大帧中间精准切掉几十至上百字节**，造成 `image1.png` 式的下半帧涂抹与 `image2.png` 式的 P 帧参考错位撕裂；而 RTSP 通常直接走 H.264/RTP（不经 TS 解析），Wireshark 在网卡层抓包无丢包，VLC 不使用 live555 的 TS 解析器，故均正常。

2. **`StreamParser::ensureValidBytes1()` 换页边界判断逻辑本身存在缺陷**：
   - `StreamParser.cpp:129` 原判据为 `if (fCurParserIndex + numBytesNeeded > BANK_SIZE)`，但第 155 行实际写入空间是 `BANK_SIZE - fTotNumValidBytes`（从 `fTotNumValidBytes` 而非 `fCurParserIndex` 开始写入）。
   - 当 `fTotNumValidBytes > fCurParserIndex`（例如 `test4Bytes()` 剩 1–3 字节或 TS 包跨边界）时，即便输入源正确返回了 `maxFrameSize()`，仍可能出现 `fCurParserIndex + maxInputFrameSize <= BANK_SIZE` 但 `fTotNumValidBytes + maxInputFrameSize > BANK_SIZE`，导致 `maxNumBytesToRead < maxInputFrameSize` 依然发生截断丢字节。
   - 此外，`StreamPuller.cpp:33` 中 `MAX_TS_FRAME_SIZE` 写死为 `7 * 188`（1316 字节），若存在大于 7 个 TS 包的 UDP/RTP 数据报（如 IP 分片或巨帧），在第 1 级 `MPEG2TransportStreamParser` 也会发生截断。

---

### 根因二（核心）：UDP Socket 接收缓冲区 `SO_RCVBUF` 仅 50 KB，与同线程同步解码叠加引发内核态突发丢包

1. **缓冲区过小**：
   - `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp:76`、`BasicUDPSource.cpp:32` 仅请求 `increaseReceiveBufferTo(..., 50*1024)`（50 KB）；`MediaSession.cpp:844` 默认下限也是 50 KB；`StreamPuller::runRTP()` 创建 `Groupsock` 后未增大 `SO_RCVBUF`。
   - Windows 默认 UDP `SO_RCVBUF` 仅 64 KB（Linux 默认 `rmem_default` 约 208 KB，扣除 `sk_buff` 元数据后有效载荷约 100 KB），请求 50 KB 几乎不起任何作用。
2. **同线程同步解码放大突发丢包**：
   - `StreamPuller` 中 live555 事件循环与 `VideoDecoder::processPacket()`（FFmpeg 解码 + `sws_scale`）在**同一线程同步执行**。解码一帧高分辨率图像耗时数毫秒至十余毫秒，期间无法调用 `recvfrom()`。
   - 尤其在 MP2T 路径下，`H264VideoStreamFramer` 必须等到**下一帧（Frame N+1）起始码到达时**才会结束并交付上一帧（Frame N）——也就是说，Frame N 的同步解码**恰好发生在 Frame N+1 整帧 UDP 突发包（直连网口 1–2 ms 内连续发出 80–300 KB）涌入内核的瞬间**！
   - 静止画面帧体积小（< 30–50 KB），不会溢出；**画面变化/关键帧体积暴涨时直接撑爆 50–64 KB UDP 接收缓冲，内核静默丢弃尾部数据包**。
   - Wireshark 在 NIC/过滤驱动层（`SO_RCVBUF` 之前）抓包故显示 0 丢包；VLC 显式将 live555 `SO_RCVBUF` 扩大至 2 MB（`2000000`）且收包与解码解耦，故无丢包；RTSP（使用 TCP 流控，或在 H.264/RTP 下于帧间 33 ms 空窗期完成解码，且可能带 SDP `b=AS:`）亦不易触发。

---

### 根因三（核心）：`StreamSink` 未按完整 Access Unit（AU）送帧，多 Slice H.264/H.265 帧触发 FFmpeg 错误隐藏与撕裂

- `src/VideoDecoder.cpp:166` 因 Linux DRM 兼容问题注释掉了 `codecCtx->flags2 = AV_CODEC_FLAG2_CHUNKS`，导致 FFmpeg 要求每次 `avcodec_send_packet()` 接收一个完整的 Access Unit（完整图像的所有 Slice）。
- 然而 `src/StreamSink.cpp:34-61` 原实现每收到 live555 交付的**单个 NALU** 就直接调用 `fCallback(&packet)` -> `avcodec_send_packet()`。
- 当 IPC 摄像头在画面变化/大帧时将一帧拆分为多个 Slice（无论经由 `H264VideoRTPSource` 还是 `H264VideoStreamFramer`），每个 Slice 都会被 FFmpeg 当作一帧独立图像解码：Slice 1 缺下半部被错误隐藏涂抹，Slice 2 缺上半部复用旧帧，引发严重的底部涂抹、运动物体跨 Slice 撕裂与画面抖动。
- 此外，原 `StreamSink` 未在 `AVPacket` 末尾保证 `AV_INPUT_BUFFER_PADDING_SIZE` 字节清零，且 `VideoDecoder` 中 `EXTRADATA_MAX_SIZE` 仅 128 字节且未预留尾部 padding。

---

### 根因四（关联边界缺陷）：live555 RTP 聚合包状态与 TS 解析细节缺陷

1. **`H264VideoRTPSource.cpp` / `H265VideoRTPSource.cpp`**：
   - `processSpecialHeader()` 处理 STAP-A/B、MTAP16/24（H.264 case 24–27）和 AP（H.265 case 48）时，未设置 `fCurrentPacketBeginsFrame = fCurrentPacketCompletesFrame = True`。当聚合包（如 SPS+PPS）紧随 FU-A 之后且前序出现丢包或异常分片时，会残留 `fCurrentPacketBeginsFrame == False`，导致聚合包被误丢弃或错误拼接。
   - `H264BufferedPacket::nextEnclosedFrameSize()` 与 `H265BufferedPacket::nextEnclosedFrameSize()` 移动 `framePtr` 跳过长度/头字段后未同步扣减 `dataSize`，存在边界越界风险。
2. **`MPEG2TransportStreamParser` & `StreamPuller::createTransportStream()`**：
   - `PIDState_STREAM` 在 `CRP_MODIFY` 下未将 `streamSink` 初始化为 `NULL`；且当 TS 流中包含不支持的音频/私有数据 PID 时，`createTransportStream()` 直接返回而保留了 `pidState->streamSource != NULL`，`processStreamPacket()` 因 `!streamSource->isCurrentlyAwaitingData()` 永久返回 `False` 导致 TS 解析死锁。
   - `MPEG2TransportStreamParser::parse()` 在 `adaptation_field_control == 0` 或 `2`（无 payload）且 `numDataBytes > 0` 时误将数据送入 `processDataBytes()`；被拒的误码/加扰包未跳过包内剩余字节；`parsePESHeader()` 对 `ES_rate_flag` 误跳 6 字节（ISO 13818-1 规范为 3 字节）。
   - `H264or5VideoStreamParser::parse()` 在第 1077 行提前清零 `fHaveSeenFirstByteOfNALUnit`，而第 1165 行 `testBytes(firstBytesOfNextNALUnit, 3)` 可能抛出 `NO_MORE_BUFFERED_INPUT` 导致重入时 `fFirstByteOfNALUnit` 被起始码首字节 `0x00` 覆盖。

---

## 3. 修改方案与涉及代码路径

所有修复均保持**单缓冲区、零额外帧延迟**的设计原则（不引入抗抖动多帧缓冲），在 `thirdparty/live555` 中的改动均以 `#if CRP_MODIFY` 标记，与项目现有约定一致：

### 3.1 `thirdparty/live555` 修改

1. **`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh` & `.cpp`**
   - 覆写 `virtual unsigned maxFrameSize() const`，返回 `184`（`TRANSPORT_PACKET_SIZE - 4`），使下游 `H264or5VideoStreamParser` 在 bank 剩余空间不足一个 TS payload 时正确触发换页，彻底消除每 150 KB 截断丢字节问题。
2. **`thirdparty/live555/liveMedia/StreamParser.cpp`**
   - 在 `StreamParser::ensureValidBytes1()` 中将换页条件修正为：
     `if (fCurParserIndex + numBytesNeeded > BANK_SIZE || fTotNumValidBytes + maxInputFrameSize > BANK_SIZE)`
     确保写指针处剩余空间 `BANK_SIZE - fTotNumValidBytes` 始终不小于 `maxInputFrameSize`，彻底消除 `fTotNumValidBytes > fCurParserIndex` 时的边界截断。
3. **`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`、`MediaSession.cpp`、`BasicUDPSource.cpp`**
   - 将默认调用的 `increaseReceiveBufferTo` 下限从 `50 * 1024` 提升至 `2 * 1024 * 1024`（2 MB）。`GroupsockHelper::increaseBufferTo()` 内部自带折半自适应回退机制，可兼容各平台 OS 上限且不增加正常播放延迟。
4. **`thirdparty/live555/liveMedia/H264VideoRTPSource.cpp` & `H265VideoRTPSource.cpp`**
   - 为 H.264 聚合包（case 24–27）与 H.265 聚合包（case 48）显式设置 `fCurrentPacketBeginsFrame = fCurrentPacketCompletesFrame = True`。
   - 在 `nextEnclosedFrameSize()` 中移动 `framePtr` 后同步扣减 `dataSize`。
5. **`thirdparty/live555/liveMedia/MPEG2TransportStreamParser.cpp` & `MPEG2TransportStreamParser_STREAM.cpp`**
   - 丢弃误码/加扰 TS 包时跳过包内剩余字节；对 `(adaptation_field_control & 1) == 0`（无 payload）的 TS 包直接跳过剩余字节而不送入 `processDataBytes()`。
   - 修正 `parsePESHeader()` 中 `ES_rate_flag` 跳过的字节数为 `3`；在 `PIDState_STREAM` 构造函数的 `#else`（`CRP_MODIFY`）分支中将 `streamSink` 初始化为 `NULL`。
6. **`thirdparty/live555/liveMedia/H264or5VideoStreamFramer.cpp`**
   - 将 `fHaveSeenFirstByteOfNALUnit = False;` 移至 `testBytes(firstBytesOfNextNALUnit, 3)` 之后、`setParseState()` 之前，防止因 `NO_MORE_BUFFERED_INPUT` 异常重入导致 `fFirstByteOfNALUnit` 被覆盖。
7. **`thirdparty/live555/liveMedia/include/RTPSource.hh`**
   - 在 `#if CRP_MODIFY` 下将 `curPacketRTPTimestamp()` 设为 `public`，供 `StreamSink` 在按 AU 组帧时准确识别 RTP 时间戳切换（不受 RTCP SR 引起的 `presentationTime` 跳变影响）。

### 3.2 `src/` 修改

1. **`src/StreamPuller.cpp`**
   - 在 `runRTP()` 创建 `Groupsock rtpSocket` 后立即调用 `increaseReceiveBufferTo(*environment, rtpSocket.socketNum(), 2 * 1024 * 1024)`；首包探测缓冲改用堆上 `std::vector<unsigned char>(0xFFFF)` 并校验 `packetSize >= 12`。
   - 将 `MAX_TS_FRAME_SIZE` 调整为 `0xFFFF`（与 `BasicUDPSource::maxFrameSize()` 及 `BufferedPacket` 的 `MAX_PACKET_SIZE` 对齐），避免单 UDP 包携带超过 7 个 TS 包时被 `StreamParser` 截断。
   - 在 `createTransportStream()` 遇到不支持的音视频/数据 PID 时关闭并置空 `pidState->streamSource`，防止 `MPEG2TransportStreamParser` 死锁；同时调整 `runRTP()` 退出时的关闭顺序为先 `demuxer` 后 `source`。
2. **`src/StreamSink.h` & `src/StreamSink.cpp`**
   - 为 H.264/H.265 视频流增加**零额外延迟的 Access Unit 组装逻辑**：
     - 对 `RTPSource`（`H264VideoRTPSource` / `H265VideoRTPSource`）：按同一 RTP 时间戳聚合非 VCL 参数集及同一帧的多个 Slice，遇到 RTP Marker 位（`M == 1` 且已含 VCL）时**立即 flush**（对标准单 Slice 或多 Slice 流零额外延迟）；若发送端漏设 `M` 位或末片丢包，则在 RTP 时间戳变化或新 AU 起始非 VCL NALU 到来时自动 flush 上一帧。
     - 对 `H264VideoStreamFramer` / `H265VideoStreamFramer`（MP2T 路径）：每次 `continuePlaying()` 前清零 `pictureEndMarker()`，聚合属于同一帧的所有 NALU，在 `pictureEndMarker() == True`（且已含 VCL）时**立即 flush**，并以新 AU 起始 NALU 兜底防粘包。
   - 在 `flushAccessUnit()` 及普通音视频包回调前，确保 `AVPacket` 数据末尾至少有 `AV_INPUT_BUFFER_PADDING_SIZE` 字节清零，并修正 `packet.pts` 计算中的 32 位 `tv_sec * 1000` 溢出风险。
3. **`src/VideoDecoder.cpp`**
   - 将 `EXTRADATA_MAX_SIZE` 扩大至 `1024`，并使用 `av_mallocz(EXTRADATA_MAX_SIZE + AV_INPUT_BUFFER_PADDING_SIZE)` 分配以符合 FFmpeg 零填充契约；在 `H264VideoDecoder` / `H265VideoDecoder` 提取 NALU 类型前补充 `offset + marker_size >= packet->size` 越界保护。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Gemini 4 Neon (XHigh)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 通过（24m46s）；16 文件（+301 / -21）；pcap1 `956 / 0 / 0`，pcap2 `845 / 0 / 0`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
