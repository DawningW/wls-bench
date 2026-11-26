# Stage 1 报告：RTP 组播画面马赛克 / 撕裂（1-crp）

> 阶段 1 只做了静态分析：没有编译、没有运行，也没有解析 pcapng。下面的结论来自读源码和看截图。

## 1. 现象

- 截图 1 红框里，画面下方大约三分之一被抹成竖条纹和块状马赛克，上方正常。截图 2 里手的上半部分是新画面，下方从一条**水平线**开始变成旧内容或马赛克。
- 这两种都是典型的 **H.264 slice（条带）数据缺失后被 FFmpeg 错误隐藏（error concealment）** 留下的痕迹：缺失的宏块从上一帧复制或按运动矢量猜，所以画面静止时几乎看不出来，**画面一变化就出现马赛克，并在 slice 边界处水平"撕裂"**。
- 按 README，单缓冲区竞态和 B 帧都已排除，与截图也对得上：撕裂边界是宏块行对齐的水平线，不像渲染撕裂。
- 结论：**进入解码器的码流不完整，或者没有按完整帧交给解码器**。Wireshark 没有丢包、VLC 播放正常，说明数据在网卡层面是完整的，问题出在本程序的接收或组帧环节。

## 2. 根因分析

### 根因 A（主要）：RTP 组播 socket 接收缓冲区太小，突发包被操作系统静默丢弃

代码路径：
- `src/StreamPuller.cpp` 中的 `StreamPuller::runRTP()`：创建 `Groupsock rtpSocket(...)` 后，直接用 `H264VideoRTPSource::createNew` 或 `MPEG2TransportStreamSource` 读这个 socket，**没有设置接收缓冲区**。
- `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp` 的构造函数只调用了 `increaseReceiveBufferTo(env, socket, 50*1024)`。这个函数只会增大不会减小，而 Windows 和 Linux 的默认值通常都 ≥ 50 KB，所以这一行实际上**什么都没做**。

对比 RTSP 路径：`MediaSubsession::initiate()`（`MediaSession.cpp` 第 841-846 行）会按 SDP 中的 `b=AS` 带宽计算 `rtpBufSize` 再加大缓冲区；如果选 TCP 传输则根本不会丢包。而裸 RTP 组播没有 SDP，也就**没有这一步**。这是 RTSP 与 RTP 两条路径在代码上唯一实质性的差别。

为什么画面一变化就会丢包：
1. 解码是同步的。`StreamSink::afterGettingFrame` → `videoCallback` → `avcodec_send_packet/receive_frame` + `sws_scale`，全部在 live555 的事件循环线程里执行。解码期间没有人读 socket，后面到达的包只能排在内核缓冲区里。
2. 画面变化或 IDR 时帧很大（几十到几百 KB，即数百个 FU-A 分片），IPC 直连千兆网以线速突发发送；同时大帧解码耗时更长。
3. 缓冲区（Windows 默认约 64 KB，Linux 受 skb truesize 开销影响实际可存数据更少）被撑满后，内核**静默丢弃** UDP 数据报。
4. Wireshark 在协议栈入口（npcap/libpcap）抓包，位置在 socket 缓冲区之前，所以"抓包不丢包、pcap 能正常播放"。VLC 自己会设置大的 UDP 接收缓冲区，所以也正常。
5. live555 的 `MultiFramedRTPSource::doGetNextFrame1` 一旦发现分片 NAL 丢了包，就把整个 NAL（即整个 slice）丢掉。解码器拿不到这个 slice，只能做错误隐藏，于是出现截图中的马赛克和撕裂，并沿 P 帧参考链传播，直到下一个 IDR。

### 根因 B（共同诱因，一并修复）：按单个 NAL 送解码器，多 slice 帧被拆成多张"残缺图"

代码路径：
- `src/StreamSink.cpp` 中的 `StreamSink::afterGettingFrame`：live555 每交付**一个 NAL**，就立刻作为一个 `AVPacket` 送进 `VideoDecoder::processPacket`。
- `src/VideoDecoder.cpp`：`codecCtx->flags2 = AV_CODEC_FLAG2_CHUNKS` 被注释掉了（作者注释说它在 Linux DRM 硬解上有 bug）。

不开 `AV_CODEC_FLAG2_CHUNKS` 时，FFmpeg（4.4）的 h264 解码器在每个 packet 解码完后都会调用 `ff_h264_field_end` 结束当前图像。所以当 IPC 把一帧编成**多个 slice**（码率高、画面复杂时很常见，有的 IPC 按字节数切 slice）：
- 第 1 个 slice 被当成一整帧输出，其余宏块全部"错误隐藏"。
- 后续 slice 又被当成"缺少开头 slice 的新图像"。

结果同样是画面变化时出现马赛克，并在 slice 边界水平撕裂。另外，单独的 SPS/PPS/SEI 包每次都会让 `avcodec_send_packet` 返回错误（"no frame!"，日志里打印 "Decode Error."）。

这个问题对 RTSP 和 RTP 都存在。如果 RTSP 和组播用的是同一编码通道配置，单靠它解释不了"RTSP 正常"，所以我把它列为共同诱因而不是唯一根因。但它是实实在在的正确性缺陷，修复后不增加延迟，所以一并修了。

## 3. 修改内容（未 commit，可直接 `git diff` 查看）

### 3.1 `src/StreamPuller.cpp`
- 新增 `RTP_RECEIVE_BUFFER_SIZE`（4 MB）和辅助函数 `increaseRTPReceiveBuffer()`：
  - 调用 live555 的 `increaseReceiveBufferTo` 增大 `SO_RCVBUF`，再用 `getReceiveBufferSize` 读回实际值并打印。
  - Linux 下如果被 `net.core.rmem_max` 截断，再尝试 `SO_RCVBUFFORCE`（有特权时生效，失败也无副作用）；还不够就打印警告，提示调大 `rmem_max`。
- `runRTP()`：创建组播 `Groupsock` 后**立即**（在等待第一个包之前）调用它。H264 裸 RTP 和 MP2T-over-RTP 都覆盖到了。
- `setupNextSubsession()`：`subsession->initiate()` 成功后，对 RTSP 的 UDP RTP socket 做同样处理，让 RTSP over UDP 也更稳。

### 3.2 `thirdparty/live555/liveMedia/MultiFramedRTPSource.cpp`
- 用 `#if CRP_MODIFY` 包住（与项目已有的 live555 修改风格一致），把默认请求的接收缓冲区从 50 KB 提到 4 MB。这样所有基于 `MultiFramedRTPSource` 的 RTP 源都有足够的突发吸收能力。

### 3.3 `src/StreamSink.h` / `src/StreamSink.cpp`：按访问单元（整帧）组包
- 对 H264/H265，同一访问单元（AU）的 NAL 依次追加到**同一个** `fReceiveBuffer` 里，每个 NAL 前补 Annex-B 起始码，没有新增任何缓冲区。AU 结束时整帧作为一个 `AVPacket` 送解码器。
- AU 结束的判断（`accessUnitEnded()`）：
  - RTP 源：用 `RTPSource::curPacketMarkerBit()`。RFC 6184/7798 规定 AU 的最后一个包置 M 位，所以收到最后一个分片就立即送解，**不增加延迟**。
  - H264/H265 Framer（MPEG-TS 路径）：用 `MPEGVideoStreamFramer::pictureEndMarker()`，读取后清零。
  - 兜底：如果新 NAL 的时间戳和已缓存的不同，先把旧 AU 送出（应对不置 M 位的发送端）；缓冲区剩余不足 1 MB 时也提前送出。其他未知源仍保持每个 NAL 一个包的旧行为。
- `SINK_RECEIVE_BUFFER_SIZE` 从 1 MB 提到 4 MB，以容纳整帧（含高分辨率 IDR）。音频 sink 的行为不变（`fPendingSize` 恒为 0）。

## 4. 与 README 约束的一致性
- **单缓冲区 / 低延迟**：没有引入帧队列或抗抖动缓冲。内核 socket 缓冲区只在突发时暂存数据，解码跟得上时它是空的，不增加稳态延迟。组帧复用原有的单一接收缓冲区，并靠 M 位在最后一个分片到达时立即送解，也不增加延迟。
- **不支持 B 帧**：没有改动，`has_b_frames = 0` 和 `AV_CODEC_FLAG_LOW_DELAY` 保持不变。
- 没有开启作者注释掉的 `AV_CODEC_FLAG2_CHUNKS`（避免 DRM 硬解 bug），而是在送解码器之前组帧来解决多 slice 问题。

## 5. 风险与后续验证（阶段 2 可做）
- 我无法在阶段 1 确认 pcap 的负载类型（H264 PT 96-99 还是 MP2T PT 33），也无法确认是否多 slice。两条路径的修复都已覆盖。
- 建议验证：
  1. 编译后用 `tcpreplay` 按原始速率重放两个 pcapng 到组播地址，对比修复前后的画面，以及日志里的 "RTP socket receive buffer" 和 "Decode Error." 出现次数。
  2. Linux 上确认 `rmem_max` 是否限制了缓冲区（看日志中的警告）。
  3. 用 RTSP（UDP/TCP）回归，确认没有劣化。
- 已知边界：如果某个发送端在 AU 末尾用置了 M 位的 STAP-A 携带多个 slice，组帧会提前送出，退化为原来的逐 NAL 行为（不会比修复前更差）。

---

# Stage 2 报告：RTP 组播画面马赛克 / 撕裂（1-crp）

## 结论（一句话）
摄像头组播推的是 **MPEG-TS over RTP（PT=33）**。播放器走 live555 的 TS 解复用 → `H264VideoStreamFramer` 这条路径。解复用出来的轨道（`MPEG2TransportStreamDemuxedTrack`）**没有声明最大帧尺寸**（`maxFrameSize()==0`），导致 `H264VideoStreamFramer` 内部的 `StreamParser` 在 bank（150 KB）快满时，仍然只用 bank 末尾剩下的几个到一百多个字节去读一个 TS 包的负载（184 B）。读不下的部分被 `MPEG2TransportStreamParser::processStreamPacket()` **静默截断丢弃**。结果是 H.264 ES 大约每 150 KB（这条码流约 0.6–0.8 s）丢几十到 180 字节，被截到的帧从断点往下出现马赛克或撕裂，并沿参考链传播到下一个 IDR。

修复只有 live555 里的一处：让解复用轨道声明 `maxFrameSize = 188`。修复后，用 pcap 回放的解码结果与 FFmpeg 直接解码同一份 TS **逐帧 bit-exact 一致**。

阶段 1 的方案（加大 socket 接收缓冲 + 按访问单元组帧）**不是根因**，已全部回退（见第 5 节）。

## 1. 调试过程

所有命令都在 `docker run --rm --network=none wls-bench-env:latest` 中执行，临时文件放在 `/work/debug_tmp`。

### 1.1 抓包分析（tshark）
- 两个 pcapng 都是 `192.168.x.x:60700 → 239.192.126.128:60700` 的 UDP 组播：
  - s1：9161 个包，52.6 s，约 1.6 Mbps。
  - s2：8180 个包，约 2 Mbps。
- RTP 解析：**payload type = 33（MP2T）**。每个 RTP 包携带 6 个 TS 包（UDP 1148 B），M 位全为 0。
  - s1：无乱序、无丢包、无重复。
  - s2：开头只有 2 个包轻微乱序（相差 4 µs，live555 的重排缓冲能处理），无丢包。
- TS 中 PID 0（PAT）、0x30（PMT）、0xE0（H.264 视频）。码流参数：H.264 Baseline，1280x720，25 fps，GOP=5（I 帧约 33 KB，P 帧 2–10 KB），**每帧单 slice**，无 B 帧。
- 把 RTP 负载拼成 `.ts` 后用 `ffmpeg` 解码：除了抓包开头不完整的那个 PES 外，没有任何错误。说明码流本身是好的，与题目描述一致。

由此排除了阶段 1 的两个假设：
- 码率只有约 2 Mbps，socket 缓冲溢出不现实。
- 每帧单 slice，"按 NAL 送解码器导致多 slice 帧被拆开"也不成立。

### 1.2 复现
- 在容器里用 Linux 版 FFmpeg 编译 CoralReefPlayer（基线源码由 `git archive HEAD` 导出到 `debug_tmp/baseline`）。
- 写了 `harness.cpp`：通过 `crp_play("rtp://127.0.0.1:60700")` 播放，把每一帧 YUV420P 落盘。
- 用 `replay.py` 按抓包时间戳（2 倍速）把 UDP 负载原样发到 127.0.0.1:60700，走的是与组播完全相同的 `StreamPuller::runRTP()` 代码路径。
- 用 `compare.py` 对每帧求 md5，与 `ffmpeg` 直接解码 TS 得到的参考帧比对（与对齐方式无关：凡是参考帧中不存在的输出帧，都判为损坏帧）。

基线结果（s1）：输出 932 帧，**278 帧损坏，分成 72 段**，大约每 14–15 帧出现一次，每段持续到下一个 IDR（≤5 帧）。问题稳定复现。

### 1.3 定位
- 在 `MPEG2TransportStreamParser_STREAM.cpp` 的 `processStreamPacket()` 里临时加日志（只加在 `debug_tmp/instr` 副本中），打印截断情况。s1 回放期间出现 **64 次截断**，例如：
  ```
  TSDEMUX TRUNCATED: want 184 maxSize 4 lost 180
  TSDEMUX TRUNCATED: want 184 maxSize 107 lost 77
  TSDEMUX TRUNCATED: want 184 maxSize 114 lost 70
  ```
  截断次数与损坏段数（72，其中含开头）基本吻合，周期 ≈ 150000 B ÷ 每帧平均约 10.6 KB ≈ 14 帧，也对得上。

- 原理分析：
  - `StreamPuller::createTransportStream()` 在 `pidState->streamSource`（即 `MPEG2TransportStreamDemuxedTrack`）上套了 `H264VideoStreamFramer`。
  - Framer 的 `StreamParser::ensureValidBytes1()` 用 `fInputSource->maxFrameSize()` 决定是否提前换 bank。由于 `CRP_MODIFY` 下 `FramedSource::maxFrameSize()` 默认返回 0，parser 只在 `fCurParserIndex + 4 > BANK_SIZE` 时才换 bank。在此之前，它一直调用 `getNextFrame(&bank[fTotNumValidBytes], BANK_SIZE - fTotNumValidBytes)`，于是 bank 尾部剩下多少就只读多少。
  - `MPEG2TransportStreamParser::processStreamPacket()` 遇到 `numBytesToDeliver(184) > maxSize` 时，只拷 `maxSize` 字节，其余 `skipBytes()` 丢弃。
  - 每填满一个 150 KB 的 bank，几乎必然丢掉一个 TS 负载的一部分。
  - 项目作者在 `MPEG2TransportStreamSource::maxFrameSize()` 的注释里已经指出过同类问题（对 TS 解析器的输入），并为 RTSP 的 MP2T 源调用了 `setMaxFrameSize()`，但**漏掉了 TS 解复用器的输出轨道**（即 Framer 的输入）。

- 为什么 RTSP 和 VLC 正常：同一台 IPC 的 RTSP 推的是 H.264 ES over RTP（`H264VideoRTPSource`），不经过 TS 解复用和 `H264VideoStreamFramer`。VLC 用的是自己的 TS 解复用器。所以只有本播放器的 RTP（TS）路径出问题。
- 为什么画面变化时才明显：被截断的帧从断点开始错误隐藏，静止画面里拿上一帧填补几乎看不出来；画面一动，就表现为截图中的马赛克和撕裂。如果截断落在 I 帧里，从断点往下整片损坏，对应截图 1 下方三分之一花屏；落在 P 帧，则局部撕裂，对应截图 2 的手。

## 2. 修改（`git diff` 仅此一处）

`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.cpp`：在构造函数中加入（`#if CRP_MODIFY`，与项目对 live555 的其他修改风格一致）：

```cpp
setMaxFrameSize(TRANSPORT_PACKET_SIZE); // 188
```

并在文件内补充 `TRANSPORT_PACKET_SIZE` 的兜底定义。

正确性推导：`ensureValidBytes1()` 中有 `numBytesNeeded = max(需求, 188)`。如果 `fCurParserIndex + 188 > BANK_SIZE`，就先换 bank。换完后有 `fTotNumValidBytes < fCurParserIndex + 原需求(≤4)`，所以 `BANK_SIZE - fTotNumValidBytes > 188 - 4 = 184`，每次读取都能装下一个完整 TS 负载，不会再截断。

影响范围：
- 所有经过 TS 解复用的视频轨道（裸 RTP 组播 PT33，以及 RTSP 中 codec 为 MP2T 的会话）都受益。
- ES 形式的 RTSP/RTP 路径不受影响。
- 不新增缓冲区，不增加延迟，符合 README 的单缓冲区和低延迟约束。

## 3. 验证结果

| 场景 | 输出帧数 | 不在参考解码中的帧 | 说明 |
|---|---|---|---|
| 基线 s1 | 932 | **278（72 段，周期约 14 帧）** | 稳定复现马赛克 |
| 修复 s1 | 945 | 9（全为孤立单帧） | 见下 |
| 修复 s2 | 845 | **0** | 与参考逐帧 bit-exact |

修复后 s1 剩下的 9 帧，用 `mixcheck.py` 逐行比对发现：它们的 Y 平面完整等于相邻的下一帧参考，或由相邻两帧上下拼接，没有任何不属于参考帧的内容。这是测试 harness 在异步回调中读取单缓冲帧时，被下一帧覆盖造成的读取竞态（README 明确说明这是单缓冲区设计固有的，不属于解码损坏），而且都不再沿参考链传播。
- 修复后不再出现任何截断日志。
- 输出帧数比参考少的部分，是异步回调繁忙时跳帧，同样是设计行为。

## 4. 涉及代码路径
`StreamPuller::runRTP()`（PT=33 分支）→ `MPEG2TransportStreamSource`（SimpleRTPSource）→ `MPEG2TransportStreamDemux` / `MPEG2TransportStreamParser::processStreamPacket()` → `MPEG2TransportStreamDemuxedTrack` → `H264VideoStreamFramer`（`StreamParser::ensureValidBytes1()`，`BANK_SIZE=150000`）→ `StreamSink::afterGettingFrame()` → `H264VideoDecoder::processPacket()`。

## 5. 阶段 1 方案的处理
阶段 1 的修改已经**全部回退**，源码树中只保留上述一处修复。阶段 1 的 diff 备份在 `debug_tmp/stage1.diff`。
- 加大 socket 接收缓冲：码率约 2 Mbps，本机回放验证中丢数据发生在用户态解复用阶段，与 socket 无关。
- 按访问单元组帧：码流每帧单 slice，不是诱因。

## 6. 调试产物
`debug_tmp/` 下保存了：`rtpstat.py`、`extract_ts.py`、`harness.cpp`、`replay.py`、`build.sh`、`run.sh`、`compare.py`、`mixcheck.py`，以及各次运行日志（`base_s1.log`、`instr_s1.log`、`fix_s1.log`、`fix_s2.log`）。体积很大的 YUV 文件已删除。

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Claude Opus 5.5 (High)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 未解决（10m07s，pcap1 `952 / 108 / 0`，pcap2 `843 / 72 / 0`）；阶段 2 通过（10m15s，完整回退阶段 1 无关改动）；1 文件（+14 / -0）；pcap1 `955 / 0 / 382`，pcap2 `845 / 0 / 168`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
