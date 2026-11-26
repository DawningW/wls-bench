# CoralReefPlayer RTP 组播画面马赛克、撕裂与抖动问题根因分析及修复报告（阶段 1）

## 一、问题现象与图像取证分析

结合题目描述、`README.md` 设计约束以及两张现场问题截图（`image1.png`、`image2.png`），可以首先明确排除两类常见误判并锁定编解码输入层面的异常特征：

1. **排除单缓冲区渲染竞态与 B 帧重排问题**：
   - `README.md` 明确说明 CoralReefPlayer 为追求极致低延迟采用**单缓冲区设计**且**不支持 B 帧**，测试流为无 B 帧的 IPC 摄像头直连码流，且同一设备通过 RTSP 播放完全正常（RTSP 同样走相同的 `VideoDecoder` 单缓冲区与 `AsyncCallback` 渲染路径）。
2. **截图中的典型 H.264/H.265 码流截断 / 分片拆帧视觉特征**：
   - **`image1.png`（下半幅垂直拉丝/马赛克）**：画面上方约 60% 完全清晰，而在光栅扫描宏块行中途（左侧蓝色展板白底文字区域约 32% 宽度处可清晰看到 1 个宏块/CTU 行的阶梯状断点）突然终止，其后直到画面底部的约 40% 区域（包括完全静止的展板与桌面）呈现**垂直向下像素拉伸**。这是 FFmpeg 在解码 IDR 帧（或帧内宏块）时，因传入 `avcodec_send_packet` 的帧数据只包含前半部分宏块（多 Slice 帧仅传入首个 Slice，或帧后半部分包/字节丢失），触发 `ff_er_frame_end()` **帧内错误隐藏（Intra Error Concealment）** 将最后一行有效像素纵向延拓到底部，并将该受损帧写入参考帧队列（DPB）的典型表现。
   - **`image2.png`（红框内运动手部水平“撕裂” + 底部马赛克 + 画面抖动）**：红框内快速移动的手臂在水平 Slice 分界线处断成两截（上下半部分处于不同时刻的运动位置），同时左下方残留参考帧污染造成的块状马赛克。这是同一帧的不同 Slice 被拆成多个独立帧解码，或某一 Slice 数据缺失导致解码器从上一参考帧复制宏块进行帧间错误隐藏（Inter Error Concealment），使同一画面中混合了相邻两帧的运动目标位置，并在连续快速输出残缺帧时表现为明显的“图像抖动”与“画面撕裂”。

---

## 二、根本原因分析

通过对 `src/` 及 `thirdparty/live555/` 源码的完整静态追踪，对比 **RTSP 播放正常**、**Wireshark 抓包零丢包**、**VLC 播放 RTSP/RTP 均正常** 三个关键事实，定位到以下三处直接导致该问题的根因：

### 根因 1：多 Slice 视频帧（Access Unit）未按完整帧聚合即逐 NALU 送入解码器（零网络丢包下亦必现）

- **涉及代码**：
  - `src/VideoDecoder.cpp`（第 165–166 行）
  - `src/StreamSink.cpp`（第 34–80 行）
  - `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`（第 180–190 行）
  - `thirdparty/live555/liveMedia/H264VideoRTPSource.cpp`、`H265VideoRTPSource.cpp`、`H264or5VideoStreamFramer.cpp`
- **机理分析**：
  1. 在 `VideoDecoder::VideoDecoder` 中，作者因 Linux DRM 硬解兼容性问题注释掉了 `codecCtx->flags2 = AV_CODEC_FLAG2_CHUNKS;`（且各平台硬解码器本身也要求每个 `AVPacket` 必须包含一帧完整的 Access Unit）。当未开启 `AV_CODEC_FLAG2_CHUNKS` 且未使用 `AVCodecParserContext` 组帧时，FFmpeg `h264dec` / `hevcdec` 默认**每次 `avcodec_send_packet()` 传入的 `AVPacket` 就是一帧完整的图像（Access Unit）**，会在每个 `AVPacket` 结束时立即调用 `field_end()` 完成当前帧并对未覆盖的宏块执行 `ff_er_frame_end()` 错误隐藏。
  2. 然而在 `live555` 中，无论是 `H264VideoRTPSource` / `H265VideoRTPSource`（RTP 负载 96+）还是 `H264or5VideoStreamFramer`（RTP `PT=33` TS 解复用），每次 `getNextFrame()` **仅向 `StreamSink::afterGettingFrame()` 交付单个 NALU**，而原 `StreamSink::afterGettingFrame()` 收到单个 NALU 后立即构造 `AVPacket` 送入 `videoDecoder->processPacket()`。
  3. **为何 RTSP 正常、静止正常，仅 RTP 组播在画面变化时出现**：
     - IPC 摄像头在 RTSP 模式下通过 SDP 协商 `packetization-mode=1`，通常编码单 Slice 整帧并通过 FU-A 在 RTP 层分片，`H264VideoRTPSource` 将 FU-A 重组为完整的单 NALU 整帧后交付，故一帧仍对应一个 NALU。
     - 而在无 SDP 协商的 RTP 组播模式下（例如遵循 RFC 6184 默认无 SDP 参数时的 `packetization-mode=0` 单 NALU 模式，或组播编码配置按字节上限 `slice-max-size` 切分多 Slice 以降低单包负载）：**画面静止时** P 帧体积很小（小于单 Slice 字节阈值），一帧只有 1 个 Slice NALU，播放完全正常；**当画面发生变化时**（如手部挥动或编码大帧），单帧数据量超过阈值，编码器将同一帧切分为多个 Slice NALU（同一 RTP 时间戳，仅最后一个包置 RTP Marker 位 `M=1`）。
     - 原 `StreamSink` 将 Slice 1 单独送入 FFmpeg，FFmpeg 立即将下半幅未解码区域按错误隐藏拉丝输出（`image1.png`）；紧接着 Slice 2（`first_mb_in_slice > 0`）作为“新的一帧”送入 FFmpeg，上半幅被错误隐藏为旧帧内容、下半幅解码为新帧内容（`image2.png` 手部撕裂 + 连续多倍残缺帧引发画面剧烈抖动）。全过程网络层**零丢包**，Wireshark 抓包与按 RTP Marker / Access Unit 组帧的 VLC 均完全正常。

### 根因 2：UDP 套接字内核接收缓冲区（`SO_RCVBUF`）仅 50 KB，叠加单线程同步软解/缩放阻塞收包导致内核态突发丢包

- **涉及代码**：
  - `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`（第 76 行、第 229–347 行）
  - `thirdparty/live555/liveMedia/BasicUDPSource.cpp`（第 32 行）
  - `thirdparty/live555/liveMedia/MediaSession.cpp`（第 843–846 行）
  - `src/StreamPuller.cpp`（第 75–95 行、第 301 行）
- **机理分析**：
  1. `live555` 的 `MultiFramedRTPSource` 构造函数中硬编码 `increaseReceiveBufferTo(env, RTPgs->socketNum(), 50*1024)`（仅 **50 KB**），`StreamPuller::runRTP()` 创建 `Groupsock rtpSocket` 后也未增大 `SO_RCVBUF`。相比之下，RTSP 模式既可走 TCP，又会在 `MediaSubsession::initiate()` 中依据 SDP 的 `b=AS:` 带宽字段计算缓冲大小；而 VLC 的 `live555`/RTP 模块则显式将接收缓冲区扩大至 **2 MB**。
  2. 在 `StreamPuller` 中，`videoCallback` 在 `live555` **同一个单线程事件循环**内同步执行 `videoDecoder->processPacket()`（包含 FFmpeg 解码与 `sws_scale` 图像缩放/格式转换），单帧耗时可达数毫秒至十余毫秒，期间完全停止读取 UDP Socket；并且 `MultiFramedRTPSource::networkReadHandler1()` 每轮 `select()` 仅调用一次 `recvfrom()` 读取单个包（而 `RTPInterface` 的 TCP 模式每轮循环最多读取 2000 次）。
  3. **为何静止正常、画面变化时坏帧且 Wireshark 抓包无丢包**：
     - 画面静止时单帧仅数 KB，远小于 50 KB；**画面变化时** IPC 编码出的大 P 帧或 IDR 帧突发体积激增至 80 KB–300 KB 以上，在千兆/百兆直连网线上以微秒至毫秒级突发到达，瞬间溢出 50 KB（Windows 默认 64 KB）的内核 `SO_RCVBUF`，导致帧后半部分 RTP 包被操作系统内核静默丢弃。
     - **Wireshark（libpcap/Npcap）工作在网卡驱动层（进入内核 Socket `SO_RCVBUF` 之前）**，因此 Wireshark 捕获到的 `.pcapng` 包完整无缺、零丢包。

### 根因 3：RTP `PT=33`（MPEG-2 TS over RTP）解复用轨道缺失 `maxFrameSize()` 及 `StreamParser` 换页条件缺陷，导致每 150 KB 静默截断丢字节

- **涉及代码**：
  - `thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh`
  - `thirdparty/live555/liveMedia/StreamParser.cpp`（第 129 行）
  - `thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp`（第 53–62 行、第 285–291 行）
  - `src/StreamPuller.cpp`（第 33 行、第 964–994 行）
- **机理分析**：
  1. 许多 IPC 摄像头的 RTP 组播输出采用 `PT=33`（`video/MP2T`，这也是 VLC 无需外部 SDP 文件即可直接通过 `rtp://@ip:port` 播放的标准组播负载格式，而 RTSP 则走 `PT=96` H264/H265 不经过 TS 解复用）。
  2. 项目作者在适配 TS 流时，虽然注意到了外层 `MPEG2TransportStreamSource` 需要指定 `maxFrameSize()` 以保证 `MPEG2TransportStreamParser`（第一个 `StreamParser`）正常切换双缓冲 Bank（见 `StreamPuller.cpp` 第 66–69 行注释），但遗漏了内层 `H264VideoStreamFramer` / `H265VideoStreamFramer` 内部的第二个 `StreamParser`（`H264or5VideoStreamParser`，`BANK_SIZE = 150000`），其输入源 `MPEG2TransportStreamDemuxedTrack::maxFrameSize()` 仍返回默认的 `0`；同时 `StreamParser::ensureValidBytes1()` 仅判断 `fCurParserIndex + numBytesNeeded > BANK_SIZE`，未考虑 `fCurParserIndex < fTotNumValidBytes` 时尾部剩余空间 `BANK_SIZE - fTotNumValidBytes < maxInputFrameSize` 的情况。
  3. 这导致每当视频基本流累计读取接近 `150,000` 字节（`fTotNumValidBytes` 落入 `[149817, 149999]`）时，`StreamParser` 不执行换页，而是向 `MPEG2TransportStreamDemuxedTrack` 请求不足 184 字节的缓冲区；`MPEG2TransportStreamParser_STREAM.cpp` 第 62 行 `skipBytes(streamSource->numTruncatedBytes())` 随即**无任何日志地静默丢弃当前 TS 包剩余的 1～183 字节码流**。画面变化码率升高时每秒跨越多次 150 KB 边界，直接在帧中部挖掉字节造成严重马赛克与错位撕裂。

---

## 三、修复方案与代码改动说明

本次修复严格遵循项目**单缓冲区、零额外帧延迟、跨平台及软硬解兼容**的设计原则，涉及以下文件：

### 1. 按完整 Access Unit（视频帧）聚合 NALU 后再送入解码器
- **`src/StreamSink.h`、`src/StreamSink.cpp`**：
  - 将接收缓冲区 `SINK_RECEIVE_BUFFER_SIZE` 扩展至 `4 MB` 并附带 `AV_INPUT_BUFFER_PADDING_SIZE` 零填充保护区，同时修复 64 位 Windows（LLP64）下 `presentationTime.tv_sec * 1000` 的 32 位 `long` 溢出。
  - 对 H.264 / H.265 流在 `StreamSink` 中将同一 Access Unit 的全部 NALU（如 SPS/PPS/VPS/SEI 及同一帧的所有 VCL Slice）连续拼接为带起始码的单个 `AVPacket`：
    - 当数据源为 `RTPSource` 时：一旦已缓存至少一个 VCL Slice 且当前 RTP 包携带 Marker 位（`curPacketMarkerBit() == True`），**立即零延迟交付整帧**；若因网络丢包丢失尾部 `M=1` 包，则在检测到下一帧 RTP 时间戳变化或新 Access Unit 起始 NALU（`first_mb_in_slice == 0` / `first_slice_segment_in_pic_flag == 1` 或 AUD/SPS/PPS/VPS/Prefix-SEI）时自动先刷出上一帧再缓存新帧，绝不混帧。
    - 当数据源为 `H264/H265VideoStreamFramer`（MPEG-2 TS 路径）时：结合 `pictureEndMarker()` 与新 Access Unit 起始检测完成整帧聚合交付。
- **`thirdparty/live555/liveMedia/include/RTPSource.hh`**：
  - 在 `CRP_MODIFY` 下公开 `RTPSource::curPacketRTPTimestamp()` 接口供 `StreamSink` 精确判定同一 Access Unit 的时间戳边界。
- **`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`**：
  - 当聚合包（STAP-A / MTAP / H.265 AP）内尚有未取完的内嵌 NALU（`nextPacket->hasUsableData()` 为真）时，将中间 NALU 的 `fCurPacketMarkerBit` 置为 `False`，仅在该 RTP 包最后一个内嵌 NALU 上保留 `M` 位（符合 RFC 6184 §5.1 与 RFC 7798 §4.1），避免聚合包提前刷出。
- **`thirdparty/live555/liveMedia/H264VideoRTPSource.cpp`、`H265VideoRTPSource.cpp`**：
  - 为 H.264 STAP-A/STAP-B/MTAP（type 24–27）及 H.265 AP（type 48）显式设置 `fCurrentPacketBeginsFrame = fCurrentPacketCompletesFrame = True`，防止前序分片包状态残留导致聚合包被误丢弃或挂起。

### 2. 扩大 UDP 套接字接收缓冲并在事件循环中批量排空内核 UDP 队列
- **`thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`、`BasicUDPSource.cpp`、`MediaSession.cpp` 及 `src/StreamPuller.cpp`**：
  - 将 RTP/UDP 套接字默认接收缓冲区（`SO_RCVBUF`）从 `50 KB` 提升至 **`2 MB`（`2 * 1024 * 1024`）**，并在 `StreamPuller::runRTP()` 创建 `rtpSocket` 后第一时间完成扩容；同时将首包探测缓冲扩大至 64 KB 并校验 RTP v2 包头，动态负载类型支持范围扩展至标准 `96..127`。
  - 在 `MultiFramedRTPSource::networkReadHandler1()`（非 Emscripten 的非阻塞 UDP 模式下）增加批量读取循环（单步最多读取 512 个包直至 `EAGAIN` 空队列），在进入 `doGetNextFrame1()` 触发同步解码与 `sws_scale` 阻塞之前，先将内核 UDP 缓冲区内已到达的数据包全部排空至用户态 `ReorderingPacketBuffer`，即使在受 `net.core.rmem_max` 限制的 Linux 系统上也能彻底消除大帧突发导致的内核态丢包。

### 3. 修复 MPEG-2 TS（`PT=33`）双缓冲换页截断与异常轨道阻塞
- **`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.hh`**：
  - 重写 `maxFrameSize() const { return 188; }`，使下游 `H264or5VideoStreamParser` 正确获知单次最大输入长度。
- **`thirdparty/live555/liveMedia/StreamParser.cpp`**：
  - 将换页判定增强为 `fCurParserIndex + numBytesNeeded > BANK_SIZE || fTotNumValidBytes + maxInputFrameSize > BANK_SIZE`，严格保证 `BANK_SIZE - fTotNumValidBytes >= maxInputFrameSize`，彻底消除每 150 KB 发生的 TS 负载截断丢字节。
- **`thirdparty/live555/liveMedia/H264or5VideoStreamFramer.cpp`**：
  - 每次成功解析 NALU 时同步更新 `fPictureEndMarker = thisNALUnitEndsAccessUnit`，并将 `fHaveSeenFirstByteOfNALUnit = False` 移至 `testBytes()` 预读下一 NALU 头部之后，防止跨包异常重入时冲掉当前 NALU 类型。
- **`thirdparty/live555/liveMedia/MPEG2TransportStreamParser_STREAM.cpp`、`src/StreamPuller.cpp`**：
  - 初始化 `PIDState_STREAM::streamSink` 为 `NULL`；当 TS 流包含非视频 PID（如音频或私有数据轨道）时，及时关闭并置空 `pidState->streamSource` 以跳过无关轨道，避免阻塞视频流解析或析构野指针；调整 `runRTP()` 退出时 `demuxer` / `sink` / `source` 的析构顺序。
- **`src/VideoDecoder.cpp`**：
  - 将 `EXTRADATA_MAX_SIZE` 扩展至 `4096` 并使用 `av_mallocz` 分配带 `AV_INPUT_BUFFER_PADDING_SIZE` 零填充的 `extradata` 缓冲区，同时在首帧关键帧完成解码器打开后立即跳出起始码扫描循环以降低首帧开销。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Gemini 4 Carbon (High)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 通过（47m40s）；14 文件（+283 / -40）；pcap1 `956 / 0 / 0`，pcap2 `845 / 0 / 0`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
