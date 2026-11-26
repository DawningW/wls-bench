# CoralReefPlayer RTP 组播画面撕裂与马赛克问题静态分析报告（阶段 1）

## 1. 问题现象与关键线索
- **现象**：同一 IPC 摄像头直连（无网络干扰、码流无 B 帧），使用 RTSP 协议播放正常；使用 RTP 组播（`rtp://...`）在**画面变化时**出现明显马赛克、下半部分竖向拉丝/花屏（`image1.png`）及运动区域撕裂（`image2.png`）。
- **外部验证**：Wireshark 抓包确认网络层无丢包；VLC 播放 RTSP 与 RTP 均无撕裂问题。
- **架构约束**：项目为追求极致低延迟采用单缓冲区设计（`VideoDecoder::outFrame`），且不包含抗抖动多缓冲区。

## 2. 根因分析

### 2.1 核心原因：RTP 组播 UDP Socket 接收缓冲区（`SO_RCVBUF`）过小导致内核层突发丢包
1. **live555 `MultiFramedRTPSource` 默认接收缓冲区仅 50 KB**：
   - 在 `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`（第 76 行）中，`MultiFramedRTPSource` 构造函数仅调用：
     ```cpp
     increaseReceiveBufferTo(env, RTPgs->socketNum(), 50*1024);
     ```
   - `H264VideoRTPSource`（用于 PT 96–99）和 `MPEG2TransportStreamSource`（继承自 `SimpleRTPSource`，用于 PT 33）均继承自 `MultiFramedRTPSource`。
   - 50 KB 远小于系统默认 UDP 接收缓冲区（如 Windows 下默认 64 KB、Linux 下 `rmem_default` 约 208 KB），`increaseReceiveBufferTo`（`thirdparty/live555/groupsock/GroupsockHelper.cpp` 第 518–537 行）仅在目标值大于当前值时才调用 `setsockopt(SO_RCVBUF)`，因此实际上不会扩大内核 UDP 接收缓冲区。

2. **RTSP 与 RTP 组播的差异**：
   - 在 `StreamPuller::runRTSP()` 路径中，live555 通过 SDP 创建 `MediaSession`，并在 `MediaSession::initiate()`（`thirdparty/live555/liveMedia/MediaSession.cpp` 第 841–846 行）根据 SDP 中的带宽声明 `b=AS:` 计算 `rtpBufSize = fBandwidth * 25 / 2` 并调用 `increaseReceiveBufferTo`（或用户选择 `CRP_TCP` 时走 TCP 流控，不存在 UDP 内核缓冲区溢出）。
   - 在 `StreamPuller::runRTP()`（`src/StreamPuller.cpp` 第 276–363 行）路径中，没有 SDP 和 `MediaSession`，直接创建 `Groupsock rtpSocket(*environment, sessionAddress, rtpPort, 1)`，未额外增大 `SO_RCVBUF`，最终仅依赖 `MultiFramedRTPSource` 的 `50*1024`（50 KB）。

3. **同步解码阻塞事件循环 + 画面变化时的码率突发（Burst）**：
   - `StreamPuller` 中视频解码在 live555 事件循环线程内**同步执行**：
     `BasicTaskScheduler::SingleStep()` -> `MultiFramedRTPSource::networkReadHandler1()` -> `doGetNextFrame1()` -> `afterGetting()` -> `StreamSink::afterGettingFrame()` -> `videoCallback()` -> `VideoDecoder::processPacket()`（调用 `avcodec_send_packet`、`avcodec_receive_frame`、`sws_scale`）。
   - 在 `VideoDecoder::processPacket()` 解码与像素格式转换期间（数毫秒至数十毫秒），live555 事件循环被阻塞，无法及时从 `rtpSocket` 调用 `recvfrom()` 读取内核队列中的 UDP 包。
   - **静态画面**时 P 帧体积很小（几 KB），不会溢出缓冲区；**画面变化时**（如手部移动或大面积前景变化），IPC 编码出的 P 帧或 I 帧体积骤增（可达数十 KB 至数百 KB，被拆分为数十到上百个 RTP/UDP 包在微秒级突发发送），极易打满 50–64 KB（或默认 ~208 KB）的 `SO_RCVBUF`，导致**内核态 UDP 丢包**。

4. **为什么 Wireshark 无丢包、VLC 正常、且表现为撕裂/花屏**：
   - **Wireshark** 在网卡驱动/链路层（`SO_RCVBUF` 之前）抓包，因此抓到的包完整无丢包。
   - **VLC** 设置了更大的 UDP 接收缓冲区（MB 级别）并将收包与解码线程解耦，因此不丢包。
   - 当 `SO_RCVBUF` 溢出丢包后，`MultiFramedRTPSource::doGetNextFrame1()`（`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp` 第 159–176 行）与 `ReorderingPacketBuffer`（第 498 行，`fThresholdTime = 100000` us，即 100 ms）会先等待 100 ms 超时（进一步加剧后续包堆积与溢出），随后判定 `packetLossPrecededThis = True` 并丢弃当前分片帧（FU-A）的剩余分片或缺失 slice，导致 FFmpeg H.264 解码器触发错误隐藏（Error Concealment）：帧下半部分出现竖向涂抹/马赛克（`image1.png`），以及参考帧局部受损导致的运动区域撕裂（`image2.png`）。

## 3. 涉及代码路径与修改说明

### 3.1 `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`
- **位置**：`MultiFramedRTPSource::MultiFramedRTPSource(...)` 第 75–76 行。
- **修改内容**：将 RTP Socket 接收缓冲区请求大小从 `50*1024`（50 KB）提升至 `2*1024*1024`（2 MB）：
  ```cpp
  // Try to use a big receive buffer for RTP:
  increaseReceiveBufferTo(env, RTPgs->socketNum(), 2*1024*1024);
  ```
- **作用**：覆盖所有基于 `MultiFramedRTPSource` 的 RTP 源（包括 `H264VideoRTPSource`、`H265VideoRTPSource`、`SimpleRTPSource`/`MPEG2TransportStreamSource`），避免在同步解码期间因码流突发导致内核 `SO_RCVBUF` 溢出丢包。`increaseReceiveBufferTo` 内部会在系统上限不允许时自动二分降级至内核允许的最大值，跨平台安全。

### 3.2 `src/StreamPuller.cpp`
- **位置**：`StreamPuller::runRTP()` 第 301–302 行（创建 `Groupsock rtpSocket` 之后、开始读取首个 RTP 包之前）。
- **修改内容**：显式调用 `increaseReceiveBufferTo(*environment, rtpSocket.socketNum(), 2 * 1024 * 1024);`：
  ```cpp
  Groupsock rtpSocket(*environment, sessionAddress, rtpPort, 1);
  increaseReceiveBufferTo(*environment, rtpSocket.socketNum(), 2 * 1024 * 1024);
  ```
- **作用**：确保在 `runRTP()` 等待首包及创建 `H264VideoRTPSource` / `MPEG2TransportStreamSource` 之前即完成 2 MB UDP 接收缓冲区的设置，防止首帧或初始化阶段的突发包丢失。

---

# CoralReefPlayer RTP 组播画面撕裂与马赛克问题调试与修复报告（阶段 2）

## 1. 调试过程与复现定位

1. **抓包流协议与码流分析**：
   - 使用 `tshark` 分析根目录下的两个抓包文件 `rtp组播流1.pcapng`（9161 个 UDP 包）与 `RTP组播包2.pcapng`（8180 个 UDP 包），确认组播地址为 `239.192.126.128:60700`，RTP Payload Type 为 **33（MPEG-II Transport Stream, `video/MP2T`）**，无丢包（`RTP组播包2.pcapng` 起始存在少量乱序但无丢包）。
   - 提取 TS 载荷并用 `ffprobe` / `ffmpeg` 验证：视频编码为 H.264 Baseline 1280x720@25fps，无 B 帧，每帧 1 个 slice；`ffmpeg` 直接解码 `stream1.ts` 和 `stream1.h264` 在中间帧均无任何宏块错误。

2. **离线复现**：
   - 在离线 Docker 容器（`wls-bench-env:latest`, `--network=none`）中构建 `CoralReefPlayer` 并编写测试程序 `/work/debug_tmp/test_rtp`，将 `rtp组播流1.pcapng` 通过 UDP 重放至 `CoralReefPlayer`（`rtp://127.0.0.1:60700`）。
   - 成功复现了与截图一致的解码错误：
     ```
     [h264] left block unavailable for requested intra mode
     [h264] error while decoding MB 0 35
     [h264] concealing 849 DC, 849 AC, 849 MV errors in P frame
     [h264] error while decoding MB 28 33
     [h264] concealing 981 DC, 981 AC, 981 MV errors in P frame
     [h264] error while decoding MB 45 34
     [h264] concealing 884 DC, 884 AC, 884 MV errors in I frame
     ```
     （1280x720 共 45 行宏块，第 31–35 行出错并触发错误隐藏，正好对应 `image1.png` / `image2.png` 中画面底部约 1/3 区域花屏/拉丝以及运动区域撕裂。）

3. **逐字节比对定位**：
   - 对比 `StreamSink` 收到的 H.264 NALU 与直接从 `stream1.ts` 提取的 `stream1.h264`，发现多处 NALU 内部字节被截断丢失（例如第 18 个 NALU 丢失 180 字节、第 38 个 NALU 丢失 77 字节、第 58 个 NALU 丢失 70 字节）。
   - 将丢失位置回溯至 `stream1.ts` 的 TS 包，确认丢失的 77 字节和 70 字节恰好位于单个 TS 包（188 字节）的**末尾**。

## 2. 根因分析

### 2.1 核心根因：`MPEG2TransportStreamDemuxedTrack` 未重写 `maxFrameSize()`，导致 `StreamParser` 缓冲区跨 bank 边界时静默丢弃 TS 载荷字节
1. 当 RTP 组播流为 `PT=33`（MPEG2-TS）时，`StreamPuller::runRTP()` 与 `StreamPuller::createTransportStream()` 建立如下数据流：
   `MPEG2TransportStreamSource` -> `MPEG2TransportStreamDemux` -> `MPEG2TransportStreamDemuxedTrack` -> `H264VideoStreamFramer`（内部使用 `StreamParser`）-> `StreamSink`。
2. 在 `thirdparty/live555/liveMedia/StreamParser.cpp`（第 121–160 行）的 `StreamParser::ensureValidBytes1(unsigned numBytesNeeded)` 中：
   - 首先查询上游数据源的单次最大帧长：`unsigned maxInputFrameSize = fInputSource->maxFrameSize();`，并取 `numBytesNeeded = max(numBytesNeeded, maxInputFrameSize)`。
   - 若 `fCurParserIndex + numBytesNeeded > BANK_SIZE`（`BANK_SIZE = 150000`），才执行双 bank 切换；随后以 `maxNumBytesToRead = BANK_SIZE - fTotNumValidBytes` 作为 `maxSize` 调用 `fInputSource->getNextFrame(...)`。
3. `StreamPuller.cpp` 曾为 `MPEG2TransportStreamSource`（TS 解复用器的**输入**）设置了 `maxFrameSize() = MAX_TS_FRAME_SIZE`（7 * 188），但 `MPEG2TransportStreamDemuxedTrack`（TS 解复用器输出给 `H264VideoStreamFramer` 的**PES 轨道源**）继承自 `FramedSource`，**未重写 `maxFrameSize()`**，其默认返回值为 **`0`**。
4. 因此，当 `H264VideoStreamFramer` 解析到当前 bank 末尾（剩余空间不足一个 TS 包载荷 184 字节，例如仅剩 107 字节），而当前 `numBytesNeeded` 较小（如 1 或 4 字节）时，`fCurParserIndex + numBytesNeeded <= BANK_SIZE`，`StreamParser` **不会切换 bank**，而是以 `maxSize = 107` 调用 `MPEG2TransportStreamDemuxedTrack::getNextFrame()`。
5. 随后在 `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp`（第 53–62 行）中：
   ```cpp
   unsigned numBytesToDeliver = numDataBytes - pesHeaderSize; // 例如 184
   if (numBytesToDeliver > streamSource->maxSize()) {         // 184 > 107
     streamSource->frameSize() = streamSource->maxSize();
     streamSource->numTruncatedBytes() = numBytesToDeliver - streamSource->maxSize(); // 77
   }
   getBytes(streamSource->to(), streamSource->frameSize());
   skipBytes(streamSource->numTruncatedBytes()); // 静默丢弃剩余 77 字节！
   ```
   导致该 TS 包尾部字节被永久丢弃，H.264 码流中部缺块损坏，解码器在对应宏块行触发错误隐藏，产生花屏与撕裂。
6. **为何仅在 RTP 组播且画面变化时明显**：RTSP 走的是 `H264VideoRTPSource`（不经过 `MPEG2TransportStreamDemux` 与 `StreamParser`）；而 RTP 组播走 `PT=33`（MP2T），画面静止时帧体积小，每帧跨越 `150000` 字节 bank 边界的概率低，画面变化时码率与帧大小骤增，频繁在帧内撞上 `BANK_SIZE` 边界导致大面积花屏撕裂。

### 2.2 辅助加固：UDP Socket 接收缓冲区（`SO_RCVBUF`）
- 保留阶段 1 对 `MultiFramedRTPSource`（`50*1024` -> `2*1024*1024`）及 `StreamPuller::runRTP()` 中 `rtpSocket`（`2 * 1024 * 1024`）接收缓冲区的扩容修改，防止高码率突发时内核态 UDP 丢包。

## 3. 修改说明

1. **`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh`**（第 36 行）：
   - 重写 `maxFrameSize()` 返回 `188`（`TRANSPORT_PACKET_SIZE`），确保 `StreamParser::ensureValidBytes1()` 在当前 bank 剩余空间不足一个 TS 包时正确触发 bank 切换，不再截断丢弃 `processStreamPacket()` 交付的数据。
2. **`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`**（第 76 行）：
   - 将 `increaseReceiveBufferTo` 目标大小从 `50*1024` 调整为 `2*1024*1024`。
3. **`src/StreamPuller.cpp`**（第 302 行）：
   - 在 `StreamPuller::runRTP()` 创建 `Groupsock rtpSocket` 后调用 `increaseReceiveBufferTo(*environment, rtpSocket.socketNum(), 2 * 1024 * 1024);`。

## 4. 验证结果

在离线容器内重新编译 `CoralReefPlayer`，使用 `/work/debug_tmp/test_rtp` 分别完整重放两个抓包文件：
- **`rtp组播流1.pcapng`**（9161 个 UDP 包）：成功解码 839 帧，FFmpeg `error while decoding` / `concealing` 错误数：**0**，`CRP_EV_ERROR`：**0**。
- **`RTP组播包2.pcapng`**（8180 个 UDP 包）：成功解码 845 帧，FFmpeg `error while decoding` / `concealing` 错误数：**0**，`CRP_EV_ERROR`：**0**。
- `CoralReefPlayer/` Git 工作区仅包含上述 3 个文件的修复改动，无任何构建临时文件残留。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Gemini 4 Hydrogen (Medium)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 未解决（4m38s，pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`）；阶段 2 通过（6m15s）；3 文件（+3 / -1）；pcap1 `956 / 0 / 382`，pcap2 `844 / 0 / 168`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
