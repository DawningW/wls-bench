# CoralReefPlayer RTP 组播画面马赛克与撕裂问题根因分析与修复报告（阶段 1）

## 1. 问题现象分析

- 同一摄像头直连（无网络干扰）：
  - RTSP 播放正常；
  - RTP 组播在**画面变化时**出现明显马赛克、拖影与纵向涂抹/撕裂（`image1.png` 下半部分出现典型的 H.264/H.265 帧尾 slice 丢失导致的垂直错误隐藏涂抹，`image2.png` 中运动的手部与下半画面均出现花屏撕裂）；
  - Wireshark 抓包无丢包且可正常播放，VLC 播放 RTSP 与 RTP 均正常；
  - 已排除单缓冲区竞态撕裂与 B 帧因素。

上述现象表明：**网络链路本身未丢包，但在播放器接收与解复用管线内部发生了码流数据丢失/截断，且与画面运动导致的码率/单帧体积突增高度相关。**

---

## 2. 根因定位

通过静态分析 `src/StreamPuller.cpp`、`src/StreamSink.cpp`、`src/VideoDecoder.cpp` 及 `thirdparty/live555/` 源码，定位出 **两类直接导致 RTP 组播在画面变化时数据截断/丢失的根因**：

### 根因一：`MPEG2TransportStreamDemuxedTrack` 未设置 `maxFrameSize()` 且 `StreamParser::ensureValidBytes1()` 换页判定有边界缺陷，导致每 150 KB 静默截断码流

1. **涉及链路（RTP 负载类型 PT=33 MPEG-TS 组播）**：
   `Groupsock` → `MPEG2TransportStreamSource` → `MPEG2TransportStreamDemux` / `MPEG2TransportStreamParser`（继承 `StreamParser`）→ `MPEG2TransportStreamDemuxedTrack` → `H264VideoStreamFramer` / `H265VideoStreamFramer`（内部 `H264or5VideoStreamParser` 继承 `StreamParser`，`BANK_SIZE = 150000`）→ `StreamSink` → `VideoDecoder`。
2. **`MPEG2TransportStreamDemuxedTrack::maxFrameSize()` 为 0**：
   - 项目作者在 `StreamPuller.cpp`（第 64-70 行、第 702 行）已经意识到 live555 的 `StreamParser` 依赖输入源的 `maxFrameSize()` 来触发双 bank 切换，因此为 `MPEG2TransportStreamSource` 设置了 `MAX_TS_FRAME_SIZE`（`7 * 188 = 1316`）。
   - 但在 `StreamPuller::createTransportStream()`（第 964-985 行）中，作为 `H264VideoStreamFramer` / `H265VideoStreamFramer` 输入源的 `pidState->streamSource`（即 `MPEG2TransportStreamDemuxedTrack`）**未设置 `maxFrameSize()`**，其返回值沿用基类默认值 `0`。
   - 在 `thirdparty/live555/liveMedia/StreamParser.cpp` 的 `StreamParser::ensureValidBytes1(numBytesNeeded)` 中：
     ```cpp
     unsigned maxInputFrameSize = fInputSource->maxFrameSize(); // 0
     if (maxInputFrameSize > numBytesNeeded) numBytesNeeded = maxInputFrameSize;
     if (fCurParserIndex + numBytesNeeded > BANK_SIZE) { /* swap banks */ }
     unsigned maxNumBytesToRead = BANK_SIZE - fTotNumValidBytes;
     fInputSource->getNextFrame(&curBank()[fTotNumValidBytes], maxNumBytesToRead, ...);
     ```
   - 当 `H264or5VideoStreamParser` 的 `fTotNumValidBytes` 增长到 `[149817, 149996]` 区间时，由于 `numBytesNeeded` 仅为 4（如 `test4Bytes()`），`fCurParserIndex + 4 <= 150000`，**不会触发 bank 切换**，导致传给 `MPEG2TransportStreamDemuxedTrack::getNextFrame()` 的 `maxSize = 150000 - fTotNumValidBytes` 小于单个 TS 包负载（最大 184 字节）。
   - 此时 `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp`（第 53-62 行）会执行截断并直接丢弃剩余字节：
     ```cpp
     if (numBytesToDeliver > streamSource->maxSize()) {
       streamSource->frameSize() = streamSource->maxSize();
       streamSource->numTruncatedBytes() = numBytesToDeliver - streamSource->maxSize();
     }
     getBytes(streamSource->to(), streamSource->frameSize());
     skipBytes(streamSource->numTruncatedBytes()); // 永久丢弃 H.264/H.265 码流字节！
     ```
   - 而 `StreamParser::afterGettingBytes()` 忽略了 `numTruncatedBytes`。因此**每消费 150 KB 视频 ES 数据就会在 bank 边界静默丢失多达 180 字节码流**。
   - 画面静止时码率低，较少落在大帧中部；画面变化（挥手等）时码率激增，150 KB 边界在每秒内多次落入 I 帧/大 P 帧中部，直接破坏熵编码数据，造成下半帧垂直涂抹与马赛克撕裂。
3. **`StreamParser::ensureValidBytes1()` 的 bank 切换判定边界缺陷**：
   - 即使输入源设置了 `maxFrameSize()`（例如 `MPEG2TransportStreamSource` 的 `1316` 字节，或 `MPEG2TransportStreamDemuxedTrack` 的 `188` 字节），`StreamParser.cpp` 第 129 行只检查了 `fCurParserIndex + numBytesNeeded > BANK_SIZE`，而第 155 行读取上限却是 `BANK_SIZE - fTotNumValidBytes`。
   - 当 `fTotNumValidBytes > fCurParserIndex`（存在尚未解析完的尾部字节，如 H.264 起始码探测剩余 1-3 字节，或 TS 解析剩余最多 187 字节）且满足 `fCurParserIndex + numBytesNeeded <= BANK_SIZE < fTotNumValidBytes + numBytesNeeded` 时，仍不会触发换页，导致 `maxNumBytesToRead < maxInputFrameSize`，依然会在 bank 边界截断 RTP/TS 数据。

### 根因二：UDP 接收缓冲区（`SO_RCVBUF`）仅 50 KB，在单线程同步解码阻塞期间容易发生内核态丢包

1. **涉及路径**：
   - `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp` 第 76 行仅调用 `increaseReceiveBufferTo(env, RTPgs->socketNum(), 50*1024);`。
   - `src/StreamPuller.cpp` 的 `runRTP()` 创建 `Groupsock rtpSocket` 后未增大接收缓冲区。
2. **触发机制**：
   - CoralReefPlayer 在 live555 事件循环线程中**同步执行** `videoCallback` → `VideoDecoder::processPacket()`（含 FFmpeg 解码与 `sws_scale` 色彩空间转换）。
   - 在解码上一帧的数毫秒至十余毫秒内，事件循环无法及时从 UDP socket 读取数据。
   - 画面剧烈变化时，高清 IPC 摄像头的 I 帧或大 P 帧体积迅速膨胀至 100 KB–300+ KB，并在极短时间内突发大量 UDP 包。
   - 50 KB（或 Windows 默认 64 KB）的 socket 接收缓冲区会被瞬间打满并由操作系统内核静默丢弃后续 UDP 包。
   - Wireshark 在网卡驱动层抓包（位于 socket 缓冲区之前），因此抓包无丢包；VLC 使用更大的 UDP 接收缓冲区（1–2 MB）且收包与解码解耦，因此 VLC 播放无撕裂。

---

## 3. 修改说明

共修改 4 个文件（所有 `thirdparty/live555` 修改均使用项目既有的 `#if CRP_MODIFY` 宏隔离）：

1. **`src/StreamPuller.cpp`**
   - 在 `StreamPuller::runRTP()` 创建 `rtpSocket` 后调用 `increaseReceiveBufferTo(*environment, rtpSocket.socketNum(), 2 * 1024 * 1024);`，将 UDP 接收缓冲区扩大至 2 MB（超出系统上限时 `increaseReceiveBufferTo` 会自动折半降级至内核允许的最大值，且不引入额外帧缓冲延迟）。
   - 在 `StreamPuller::createTransportStream()` 中对 `pidState->streamSource` 调用 `inputSource->setMaxFrameSize(TRANSPORT_PACKET_SIZE);`，防止下游 `H264VideoStreamFramer` / `H265VideoStreamFramer` 的 `StreamParser` 在 bank 边界截断 TS 负载。

2. **`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.cpp`**
   - 在 `MPEG2TransportStreamDemuxedTrack` 构造函数中（`#if CRP_MODIFY`）调用 `setMaxFrameSize(188);`，确保所有使用 `MPEG2TransportStreamDemux` 的下游 `StreamParser` 都能获知单个 TS 包的最大交付尺寸。

3. **`thirdparty/live555/liveMedia/StreamParser.cpp`**
   - 在 `StreamParser::ensureValidBytes1()`（第 129 行，`#if CRP_MODIFY`）将换页条件由：
     `if (fCurParserIndex + numBytesNeeded > BANK_SIZE)`
     修正为：
     `if (fCurParserIndex + numBytesNeeded > BANK_SIZE || fTotNumValidBytes + numBytesNeeded > BANK_SIZE)`
     保证传给 `fInputSource->getNextFrame()` 的 `maxNumBytesToRead = BANK_SIZE - fTotNumValidBytes` 恒大于等于 `maxInputFrameSize`，彻底消除 bank 边界截断。

4. **`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`**
   - 在 `MultiFramedRTPSource` 构造函数中（第 76 行，`#if CRP_MODIFY`）将默认 `increaseReceiveBufferTo` 大小从 `50*1024` 提升至 `2*1024*1024`（2 MB），同时覆盖 RTP 组播（PT=33 与 PT=96-99）及 RTSP/SDP over UDP 场景。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Gemini 4 Neon (Medium)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 通过（8m01s）；4 文件（+14 / -0）；pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
