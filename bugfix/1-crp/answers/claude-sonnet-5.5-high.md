# Stage 1 report (1-crp): RTP multicast picture tearing / mosaic

Static analysis only (nothing compiled or run, pcapng files not parsed; screenshots and source read only). The fix is unverified.

## Symptoms
- The same camera plays correctly over RTSP and in VLC, but over RTP multicast the picture tears and shows mosaic when the scene changes.
- The tear is a horizontal band: the top part of the picture is correct and the lower part is smeared or blocky. The "hand" in the red box is cut by such a band.
- Wireshark sees no packet loss, so the network is fine.

## Code path
`StreamPuller::runRTP()` (src/StreamPuller.cpp) has two branches:
- PT 33 (MPEG-TS over RTP): `MPEG2TransportStreamSource` -> `MPEG2TransportStreamDemux` -> `createTransportStream()` -> `H264VideoStreamFramer` -> `StreamSink`.
- PT 96-99 (H264 over RTP): `H264VideoRTPSource` -> `StreamSink`.

Both end in `StreamSink` (src/StreamSink.cpp), which hands data to `VideoDecoder::processPacket()` -> `avcodec_send_packet`. I could not check the pcap, so I do not know which branch the camera uses. The fix covers both.

## Root causes
### 1. Slices of one picture were decoded as separate pictures (main cause)
- live555 delivers **one NAL unit per `afterGettingFrame`**. The old `StreamSink::afterGettingFrame` wrapped each NAL in its own `AVPacket` and called the decoder immediately.
- IPC encoders commonly encode a picture as several slices. Each slice is its own NAL (type 1 or 5, `first_mb_in_slice` != 0 for all but the first).
- FFmpeg's H.264 decoder treats one `AVPacket` as one access unit and finishes the picture at the end of each packet. Without `AV_CODEC_FLAG2_CHUNKS`, which the code leaves off, a packet starting with a slice that has `first_mb != 0` is handled as a new picture. FFmpeg then outputs a half-decoded frame and conceals the rest, which gives the band tearing and mosaic. The larger and more complex the picture, the more it shows.
- A single-slice stream shows no problem, which would also explain why RTSP and VLC look normal. I could not check this against the pcap or the RTSP stream.
- The RTP marker bit and the TS framer's end-of-access-unit logic were not used anywhere in the player.

### 2. Small kernel receive buffer for the multicast socket (secondary)
- In `runRTP()` the `Groupsock` is created directly and never gets a larger `SO_RCVBUF`. `MediaSubsession::initiate()`, used on the RTSP path, enlarges it.
- This is the only RTP-vs-RTSP difference I found at the socket level. live555's own `increaseReceiveBufferTo(50KB)` in `MultiFramedRTPSource` does nothing when the default (about 208 KB on Linux, smaller on Windows/macOS) is already larger.
- This single thread is busy decoding between reads. A picture sent as a burst (large I frame or scene change) can overflow the kernel queue silently, so Wireshark shows every packet but the application misses some. Missing slice packets also give mosaic, because FFmpeg conceals the missing slices.
- Note: on Linux the value is capped by `net.core.rmem_max`, so this fix may be limited there. I did not use `SO_RCVBUFFORCE`.

## Changes
### src/StreamSink.h / src/StreamSink.cpp
- For H264/H265, `StreamSink` now collects all NAL units of an access unit in `fReceiveBuffer`. Each NAL gets its own start code, and `getNextFrame` appends at `fFillSize + 4`. The whole AU goes to the decoder as one `AVPacket`.
- End of access unit (`isEndOfAccessUnit()`):
  - RTP sources: the RTP marker bit (`RTPSource::curPacketMarkerBit()`).
  - TS path: `MPEGVideoStreamFramer::pictureEndMarker()`. The sink gets the framer through the new `setFramer()`. live555 never clears this flag by itself, so `continuePlaying()` clears it before each new NAL.
- Fallback: if a NAL arrives with a different presentation time while a previous AU is still pending (for example the marker packet was lost), the pending AU is delivered first. The AU is also flushed when less than 256 KB of buffer is left.
- A truncated NAL discards the pending AU.
- Audio and other codecs behave as before.
- The decoder's SPS/PPS/IDR state machine already scans Annex-B packets with several NALs, so `VideoDecoder.cpp` is unchanged.

### src/StreamPuller.cpp
- `runRTP()`: `increaseReceiveBufferTo(..., 4 MB)` on the multicast socket (`RTP_RECEIVE_BUFFER_SIZE`).
- `createTransportStream()`: the H264/H265 framer is kept and passed to the sink with `setFramer()`.

## Open issues
- RTSP H264/H265 video also goes through `StreamSink`, so it now assembles access units by marker bit too. If a camera never sets the marker bit, the timestamp fallback adds about one frame of latency.
- A STAP-A packet carrying several slices with the marker set would end the AU after its first NAL. This should be rare.
- The first RTP packet of the multicast stream is consumed in `runRTP()` to detect the payload type and then dropped. This only affects the first picture and is not changed.

---

# Stage 2 report (1-crp): RTP multicast picture tearing / mosaic

## Result
The root cause is a data-dropping bug in the MPEG-TS demux. It is fixed with a one-line change in a third-party file. After the fix, both pcaps replay with every frame bit-identical to FFmpeg's reference decode and with zero decode errors.

## Process
1. Analyzed both pcapng files with tshark in the offline container.
   - Both files carry **MPEG-TS over RTP** (PT 33) on 239.192.126.128:60700, with 6 TS packets (1128 bytes) per RTP packet.
   - The video is H.264 Baseline 1280x720 at 25 fps, one slice per frame, one PES per frame, and an IDR about every 5 frames. This ruled out my stage-1 multi-slice theory.
   - Capture 1 has no seq gaps. Capture 2 has 3 gaps and 2 out-of-order packets.
2. Built the library in the container (`/work/debug_tmp`, outside the source tree).
   - The container cannot join a multicast group without a multicast-capable interface. I replayed the pcap payloads over loopback as **unicast** UDP instead.
   - A harness calls `crp_play("rtp://127.0.0.1:60700")` and prints a CRC32 of every output frame. Reference CRCs come from `ffmpeg -i <ts>`.
3. Reproduced the bug on the unmodified tree: at 1x real-time speed on capture 1, **273 of 952 output frames were corrupt** and 117 "error while decoding / concealing" log lines appeared. On capture 2 at 2x, 404 of 845 frames were corrupt.
4. Located where the data is lost.
   - In a scratch copy of the baseline, I dumped the NAL units that reach the decoder. About 64 NALs were shorter than in the original ES, with 70-180 bytes missing in the middle. The gaps were spaced about 150000 bytes apart, which equals live555's `StreamParser` `BANK_SIZE`.
   - I then logged what the TS parser delivers to the demuxed track. `processStreamPacket()` reported **64 "truncated" deliveries** (`numTruncatedBytes > 0`).

## Root cause
Path: `H264VideoStreamFramer` (a `StreamParser`, 150000-byte banks) reads from `MPEG2TransportStreamDemuxedTrack`.

- `StreamParser::ensureValidBytes1()` asks its input for `BANK_SIZE - fTotNumValidBytes` bytes, i.e. whatever space is left in the current bank.
- It only swaps banks early when `fInputSource->maxFrameSize()` is larger than the needed bytes.
- `MPEG2TransportStreamDemuxedTrack` never set a maximum frame size (`FramedSource::fMaxFrameSize` defaults to 0), so no early bank swap happened.
- `MPEG2TransportStreamParser::processStreamPacket()` copies each TS packet's payload (at most 184 bytes) into the framer's buffer at once. If the remaining bank space is smaller than the payload, it sets `numTruncatedBytes`, copies only what fits and skips the rest of the packet.
- Roughly every 150000 bytes of video, one TS packet's payload is cut at a bank boundary (70-180 bytes lost inside a NAL). That frame is decoded with missing data, so FFmpeg shows mosaic or tearing, and the error carries into the following P frames until the next clean IDR.
- A static scene produces small frames and few bank crossings. Large frames, such as scene changes or motion, are hit more often and the damage is more visible.
- RTSP does not use the TS demux, since the camera sends RTP/H.264 there, and VLC has its own demuxer. Both are therefore unaffected.
- The network and the reordering buffer were not at fault, which matches Wireshark seeing no loss.

## Fix
`thirdparty/live555/liveMedia/MPEG2TransportStreamDemuxedTrack.cpp`: in the constructor, under `#if CRP_MODIFY`, call `setMaxFrameSize(188)` with an explanatory comment. The downstream `StreamParser` then swaps banks before the free space drops below one TS packet. This is the same mechanism the project already uses for `MPEG2TransportStreamSource::maxFrameSize()`.

`git status` shows only that one file modified. My stage-1 changes (`StreamSink`, `StreamPuller`) were reverted: they were based on the wrong multi-slice hypothesis and are not needed.

## Verification
All runs use unicast replay of the pcap through `crp_play`; compare against the FFmpeg reference decode of the extracted TS.

| Build | Capture / speed | Frames out | Corrupt frames | Decode error lines |
|---|---|---|---|---|
| baseline | 1 / 1x | 952 | 273 | 117 |
| baseline | 2 / 2x | 845 | 404 | 90 |
| fixed | 1 / 3x | 956 | 0 | 0 |
| fixed | 1 / 1x | 956 | 0 | 0 |
| fixed | 2 / 2x | 845 | 0 | 0 |

The frames not output (5-6 in capture 1) are the ones before the first IDR, where the decoder waits for a keyframe. This is existing behavior.

---

## 离线双 `.pcapng` 回放实测汇总

- **模型**：Claude Sonnet 5.5 (High)
- **实测记录**（三元组为 `输出帧数 / corrupt_errors / no_frame_logs`）：阶段 1 未解决（7m31s，pcap1 `952 / 108 / 0`，pcap2 `845 / 72 / 0`）；阶段 2 通过（12m47s，完整回退阶段 1 无关改动）；1 文件（+8 / -0）；pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`
- **基线对照**：未修改原码 pcap1 `952 / 108 / 381`，pcap2 `845 / 72 / 168`；最小 1 行修复对照 pcap1 `956 / 0 / 382`，pcap2 `845 / 0 / 168`。
