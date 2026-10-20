# Performance Audit

Audit date: 2026-10-14.

## Pipeline and bottlenecks

`CADisplayLink` drives capture on the main thread. On device, `CARenderServerGetDirtyFrameCount` skips unchanged displays. A changed display is rendered at native resolution, transferred through IOSurface, then rotated/copied/scaled into the back buffer. Publishing swaps the buffers and marks the full output as modified. LibVNCServer merges pending modified regions and encodes them when a client requests an update; this is not a queue of captured frames.

The bundled headers identify LibVNCServer 0.9.15. Its [client output loop](https://github.com/LibVNC/libvncserver/blob/LibVNCServer-0.9.15/src/libvncserver/main.c) holds each client's `sendMutex` across a framebuffer update. [Encoding and flushing](https://github.com/LibVNC/libvncserver/blob/LibVNCServer-0.9.15/src/libvncserver/rfbserver.c) run synchronously; [socket writes](https://github.com/LibVNC/libvncserver/blob/LibVNCServer-0.9.15/src/libvncserver/sockets.c) wait for socket writability under backpressure. TrollVNC waits for these client locks to publish its next frame. Consequently, either encoder work or a slow receiver/network can stall capture, and one slow client can affect the others. The six former tuning controls did not change this architecture or enable hardware video encoding.

The other significant candidate is native-resolution capture itself, followed by rotation and high-quality scaling. A source comment estimating capture at about 20 ms is not a current measurement. The previous capture FPS log counted changed captures before publication, and its instantaneous FPS came from display-link cadence; neither measured client-visible FPS.

## Removed tuning controls

| Former setting / CLI | Fixed policy | Assessment of the old implementation |
| --- | --- | --- |
| Defer window / `-d` | No additional application deferral (`0`) | Bypassed when dirty detection was off. When enabled, flush depended on another changed capture; an isolated final change could remain pending indefinitely. LibVNCServer's own default 5 ms deferral remains unchanged. |
| Max in-flight / `-Q` | No counter-based frame dropping (`0`) | Counted simultaneous client update calls, including socket waits, rather than queued frames. With one client, the default `2` normally never triggered. Dropping after capture had consumed the display dirty counter could lose the final change until a later screen change. |
| Tile size / `-t` | Removed with tile detection | Former default `32` only mattered with dirty detection enabled. It changed hashing granularity, not native capture cost or encoder throughput. No unused tile-size constant is retained. |
| Fullscreen threshold / `-P` | Dirty detection disabled (`0`, existing default) | Fullscreen publication avoids the old detector's correctness problems. This preserves default behavior, but users who previously enabled useful partial updates can see higher traffic for localized changes. |
| Max rectangles / `-R` | One full-output modified region | Former default `256` was bypassed without dirty detection. The builder truncated dirty runs before bounding-box fallback, so the box could exclude changed pixels. Values above the internal 1024-entry buffer did not reliably trigger fallback. This policy does not constrain the encoder's wire rectangle count. |
| Non-blocking swap / `-a` | Off; use the existing blocking publication path | Failed try-locks fell back to writing into the front buffer while encoders could still be reading it. This trades waiting for races/tearing and extra copying, not greater encoder or network capacity. |

The old detector also compared sparse hashes against full-frame tile hashes, so unchanged tiles could be marked dirty. Pending regions were then combined with current regions redundantly; a zero-rectangle result caused a full-screen update. These mechanisms added work without reliable reduction of the encoded area. A host-side reproduction using the original functions marked all six tiles dirty for an identical image, and a separate two-distant-tile case with a one-rectangle cap omitted the second tile. These are correctness reproductions, not device throughput benchmarks.

The six settings are removed from Preferences, command-line parsing and managed-plist generation. Existing stored values are ignored. Remove the old flags from launch scripts; they are no longer accepted. Tile hashing, pending masks and unsafe asynchronous copy branches have also been removed.

This cleanup retains the existing blocking synchronization design. Its separate lock/unlock client-list traversals need a lifecycle audit for concurrent disconnects, and reverse connections use `rfbProcessEvents`, whose upstream send path does not take the same outer `sendMutex`. Disabling asynchronous swapping alone does not prove those paths race-free; a synchronization redesign is a separate change.

## Measuring a device

Debug builds now emit `update stats` per client approximately every five seconds while updates complete. `wall` includes encoding and synchronous writes; `cpu` measures CPU time on that output thread; both are averages per update call. `rfb` is the library's accounted RFB bytes per second, excluding TCP/TLS/WebSocket overhead. `encoding` is the negotiated encoding number. These counters are absent from Release builds.

Compare these with capture's `elapsed` and verbose (`-V`) `frame summary` timings. A large `publish/wait` means publication is waiting on clients. Update wall time close to thread CPU time points to CPU work; a large gap points to waiting or descheduling, and must not automatically be labelled network time. Use a fixed workload, viewer encoding/quality and resolution, then compare USB forwarding and Wi-Fi, and native size versus `-s 0.5`. A non-incremental full-frame probe measures update service capacity, not animation or end-to-end input latency.

## Device observations (2026-10-14)

Test device: iPhone 15 Pro (`iPhone16,1`), iOS 17.0, output 1180×2558. A read-only RFB probe requested full non-incremental updates for approximately seven seconds per case, consuming and discarding pixels without displaying or saving them. Tight requested quality level 6 and compression level 6. This used the device's current, mostly static screen, not a controlled animation or the VNC Viewer rendering path. Runs were sequential, with minor screen changes possible.

The temporary Debug server used this change, `-s 1 -F 30`, and USB/iproxy SSH forwarding. Repeated requests re-encode the existing framebuffer, so request rates can exceed the capture cap. Server timings below cover the first approximately five-second statistics window; probe rates cover the full run.

| Encoding | Update wall / thread CPU | Probe updates/s | Probe RFB Mbps |
| --- | --- | --- | --- |
| Tight | 7.43 / 7.20 ms | 69.31 | 129.61 |
| zlib | 51.46 / 51.45 ms | 17.15 | 28.61 |
| Raw | 195.98 / 18.12 ms | 3.51 | 339.29 |

zlib was CPU-bound in this sample. Raw spent most update time off CPU, consistent with synchronous sending to a bandwidth-limited receiver path. Tight was much cheaper to encode than zlib. Two changed-capture log samples took 17.29 and 13.41 ms before rotation/scaling/publication; they are individual observations, not an average or an animation benchmark.

A separate transport comparison used the **same original installed 3.2-272 server** for both paths:

| Encoding | USB/iproxy + SSH: updates/s, Mbps | Direct LAN: updates/s, Mbps |
| --- | --- | --- |
| Tight | 67.62, 127.00 | 32.26, 60.47 |
| Raw | 3.50, 338.51 | 2.72, 262.26 |

Tight's LAN result includes request round trips and scheduling, not just link bandwidth: Raw moved over 262 Mbps on the same LAN path. A native Raw update is approximately 12.07 MB, requiring about 2.90 Gbps at 30 full updates/s. The unchanged installed server and temporary Debug server produced similar USB rates; removing the controls is not evidence of an encoder throughput improvement.

These probe observations identify encoding CPU cost and synchronous sending as important limits, but do not establish VNC Viewer's negotiated encoding, decode/render cost, motion FPS or input latency. At this probe stage, the installed app was not replaced, and the temporary server was stopped and removed after measurement. The roothide Debug package was subsequently built and installed for the RealVNC Viewer sessions below.

## RealVNC Viewer session after installation (2026-10-14)

The installed roothide Debug server's 13:27–13:28 log confirms the user's low-frame-rate report. RealVNC Viewer initially negotiated standard ZRLE, then sent another encoding list and the server switched to Hextile (`encoding=5`). The screenshot's requested encoding was ZRLE2, but its last-used encoding and the server log both identify Hextile. The server selects the first supported encoding in the client's ordered list; it is not sending Tight in this session.

During the sustained 13:27:48–13:28:24 interval, update rates were approximately 6.3–6.6/s. Average encoding-plus-write wall times were approximately 103–125 ms, with 50–54 ms of output-thread CPU time and 156–172 Mbps of accounted RFB traffic. Changed-capture samples were approximately 8–13 ms. Both encoding CPU work and time off CPU in the synchronous update path matter here; the latter is consistent with sending/backpressure but is not a measurement of network wait alone. This session used full-output publication; the later assessment below addresses the user's predominantly full-screen mobile workload.

The earlier mostly static Tight probe is not representative of this RealVNC Viewer workload. Removing the six controls did not resolve the reported interactive performance problem.

### ZRLE with reduced color depth

The user's subsequent screenshots show `depth 6 (8 bpp) rgb222`, only 64 colors, explaining the severe posterization. The requested ZRLE2 session still reports last-used encoding ZRLE; these screenshots do not demonstrate a separate ZRLE2 encoding path. The user reported smoother motion but still poor usability. A low-color ZRLE statistics window recorded approximately 12.6 updates/s, with 37.65 ms wall time and 36.98 ms thread CPU per update. This improvement is confounded by reduced color depth and cannot be attributed to the encoding change alone.

The initial client-setting instructions omitted `ColorLevel=full`. According to the [RealVNC Viewer parameter reference](https://help.realvnc.com/hc/en-us/articles/360002254618-RealVNC-Viewer-Parameter-Reference), the full-color comparison uses `Quality=CUSTOM`, `PreferredEncoding=ZRLE`, and `ColorLevel=full`. The next screenshot confirms `depth 24 (32 bpp)` and last-used encoding ZRLE.

### ZRLE with full color

After the 13:33:24 reconnection, the installed Debug server's 13:33:29–13:34:10 statistics windows recorded:

| Metric | Observed range |
| --- | --- |
| Output resolution and client pixel format | 1180×2558, depth 24 / 32 bpp |
| Actual encoding | ZRLE (`encoding=16`) |
| Completed server updates | 2.79–3.43/s |
| Average update wall time per window | 232.66–300.88 ms |
| Average output-thread CPU time per update per window | 223.14–274.04 ms |

The user reported that full-color ZRLE was too choppy to use. CPU time accounts for most of the update duration, identifying encoding-side CPU work as the dominant limit in this session. Network throughput alone cannot explain that result. These are server update statistics, not measurements of client-rendered FPS or end-to-end input latency; the ranges describe window averages, not individual-update extrema or latency percentiles.

Source inspection of the matching LibVNCServer 0.9.15 [ZRLE encoder](https://github.com/LibVNC/libvncserver/blob/LibVNCServer-0.9.15/src/libvncserver/zrle.c) and [output stream](https://github.com/LibVNC/libvncserver/blob/LibVNCServer-0.9.15/src/libvncserver/zrleoutstream.c) found that the stream initializes zlib with `Z_DEFAULT_COMPRESSION` (level 6), without applying the client's compression-level preference. Lower compression could trade more transmitted bytes for less CPU work without reducing color depth. No lower-level benchmark or implementation change was completed, so no performance gain is claimed.

## Final assessment and stopping point

The target workload is mobile interaction: scrolling, transitions, blur and animated backgrounds frequently change most or all of the screen. Reducing the modified region is therefore not a sufficient answer to the reported problem. The earlier suggestion to prioritize partial updates did not address this workload. Removing the old dirty detector still has a traffic tradeoff for localized changes, but restoring or tuning it is not the proposed solution for sustained full-screen motion.

The observed tradeoffs are substantial: Hextile used approximately 156–172 Mbps while completing only about 6–7 updates/s; full-color ZRLE reduced the practical update rate to about 3/s with most update time spent on CPU; reduced-color ZRLE improved throughput at the cost of visibly unacceptable 64-color output. The mostly static Tight probe does not establish that the current RealVNC Viewer path can deliver a comparable interactive experience.

For this implementation, client and native-resolution full-color workload, the remaining gap is larger than the evidence supports addressing through the former six controls or another compression-level adjustment. This is not a claim that every VNC implementation performs poorly, nor a measured speedup from removing those controls. The cleanup removed ineffective controls and identified correctness problems; it did not make mobile interaction acceptably smooth.

A different direction, if reconsidered in the future, would be hardware video encoding with temporal compression, bounded transmission backlog and a separate input channel, together with a compatible client. That is a separate architecture and product decision, not a completed optimization or a promised performance result. VNC can remain useful for compatibility and occasional remote control.

At the user's request, performance tuning ends here. No further compression experiments, partial-update work, synchronization redesign or video-streaming prototype will be pursued as part of this audit. This final step updates only this document; it makes no additional code changes, builds or device installations.
