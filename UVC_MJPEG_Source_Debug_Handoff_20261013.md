# UVC 4K MJPEG 局部撕裂：源码排查与对照实验交付说明

日期：2026-10-13  
对象：Camera/JPEG、UMD、UVC gadget 相关源码开发人员  
目标：定位 Windows 收到的 MJPEG 帧为何出现局部错位和绿色区域，验证 buffer 生命周期、同步、长度或帧组装是否存在错误。

## 1. 现象与已确认事实

设备通过 UVC 向 Windows 输出 3840×2160、MJPEG、30 fps，PotPlayer 实时播放偶发局部撕裂，无明显卡顿。用户使用无 Camera override 配置，并表示 CPU/interconnect 已设最高；CPU 三组频率实测为 1804800、2419200、2841600 kHz，interconnect 未独立核实。

使用 FFmpeg DirectShow 按相同模式采集两分钟，通过 stream copy 保存，未解码重编码。采集正常退出，获得 3598 帧，平均约 29.975 fps。用户确认本地录像中的两张异常帧与此前实时看到的故障一致。

| 原始帧 PTS | JPEG 字节数 | JPEG 起止标记 | 独立软件解码结果 |
|---|---:|---|---|
| 96.512 秒（01:36.512） | 712003 | FFD8 / FFD9 | overread 8；块状错位、底部绿色区域 |
| 103.744 秒（01:43.744） | 879251 | FFD8 / FFD9 | overread 8；块状错位、底部绿色区域 |

两张独立 JPEG 的 SHA-256 与录像中对应原始压缩帧包一致。故障已存在于 host 保存的数据，不能仅由 PotPlayer 的实时显示同步解释。仍不能区分设备输出异常、传输/组帧异常或 host 接收路径问题。

其他观察：相邻压缩帧包完全相同的 hash 为 0 对；不排除不同编码字节对应重复画面。平均包间隔约 33.361 ms，最长 52 ms；不存在明显长时间断帧的证据。JPEG 起止标记正常、解码进程正常退出均不代表每帧内容正常。

## 2. 尚未证实的结论

- 未证明 JPEG reset 是根因。历史 trace 中有 cam_jpeg_enc_reset_hw，但尚无 reset reason 与坏帧关联记录。
- 未证明 USB 带宽不足或 IRQ 绑核有效。固定 irqbalance 暂停、USB CPU0 时，JPEG CPU1→CPU3→CPU1→CPU3 的人工计数为 3→1→9→6 次/60秒；波动明显，性能状态也变化。
- UMD C3-0-Handler 曾接近占满一个核心，也有运行占比下降的窗口；不能仅凭 CPU 占用判断空转、处理负载或根因。
- 历史 Camera JPEG dump 可正常解码，不能证明与 UVC 实际发送的故障帧为同一份数据。

本次工作优先获取同一帧跨边界证据，避免把提高优先级、绑核或大量日志当作修复。

## 3. 第一优先实验：独立 UVC 发送 buffer

增加可切换的调试路径，默认关闭；保留原路径用于 A/B/A 对照。

原路径可能为共享/零拷贝 buffer。实验路径要求：

```text
确认 JPEG 编码完成
→ 等待并检查输出 fence
→ 完成源 buffer 所需的 CPU 可见性/缓存同步
→ 获取一个空闲的独立 UVC 发送 buffer
→ 仅复制该帧有效 JPEG 字节
→ 完成发送 buffer 所需的设备可见性/缓存同步
→ 设置正确 bytesused 并提交 UVC
→ 保持发送 buffer 内容不变
→ 确认 UVC 不再引用该 buffer
→ 返回空闲池
```

实现要求：

1. 使用多个独立发送 buffer 构成有明确状态的池，不能每帧覆盖同一个 staging buffer。
2. 以实际 API/驱动语义确认释放时机。QBUF 成功、某个 USB request complete、用户态提交回调均不能未经验证就等同于整帧 buffer 释放。V4L2 路径应核查 DQBUF 与驱动 vb2 buffer_done 的关联。
3. 复制完成后可归还源 buffer，但前提是源端不会在复制期间复用/写入。发送 buffer 直到释放前禁止再次写入。
4. 缓存同步依具体内存分配器、DMA-BUF exporter、映射和设备访问方式实现。fence 表示完成关系，不能替代对缓存一致性的核查；不要添加平台不需要或不正确的 cache 操作。
5. 必须区分 capacity、JPEG valid size、bytesused、payload offset 和任何尾部元数据。若使用 Android JPEG BLOB，核查相关结构/位置及有效长度，不能把整个分配区发送给 host。
6. 若发送池用尽，采用显式记录的等待或丢帧策略，不能抢占仍在发送的 buffer；记录等待、丢帧和池高水位。
7. 加入长度检查：有效长度大于0且不超过源有效区域和目标容量。起止标记只作辅助检查，不作内容正确性判据。
8. stop、flush、error、disconnect 和重新协商时，同样保证 in-flight buffer 在真正释放之前不被回收。

建议状态模型：FREE → COPYING → READY → IN_FLIGHT → FREE。发生异常时记录非法转换及上下文，避免每帧打印大量日志。

结果解释：复制路径若明显减少故障，支持原路径存在生命周期、同步或时序相关问题，但复制也改变时序，不能直接宣布提前复用已证实；需结合下面的数据比较和事件记录。

## 4. 第二优先：同一帧三个位置的数据比较

| 位置 | 获取时机 | 记录/保存内容 |
|---|---|---|
| P1：JPEG 输出 | done、fence 满足且数据对读取方可见后 | frame/request ID、buffer ID、有效长度、hash，选定帧的 JPEG |
| P2：UVC 提交前 | 独立发送 buffer 内容固定且同步完成后 | 同一 frame ID、发送 buffer ID、bytesused、hash，选定帧的 JPEG |
| P3：Windows 接收后 | 一个完整 MJPEG 帧包交付采集程序时 | 接收序号、PTS、长度、hash、原始 JPEG |

注意：

- host PTS 与设备 monotonic 时间不是同一时钟；不能仅按秒数强行关联。
- 帧关联可使用调试 frame ID（如可行的 JPEG APP/COM 标记，且各路径保持一致），或采用保存的长度+hash+连续帧顺序匹配。插入标记也会改变数据，应与无标记基线对照。
- 哈希仅覆盖实际发送的 JPEG 范围，所有位置使用相同算法。单独记录平台元数据/额外封装；如路径会改写 JPEG 元数据，先确认比较范围。
- 全帧 hash/dump 可能扰动时序。优先有界内存环形存储、选择性采样或短时采集；记录开销并复测无记录条件。
- P1/P2 的 dump 必须稳定持有对应 buffer，异步写文件任务不能保存一个已归还的裸指针。

| 证据 | 下一步方向 |
|---|---|
| P1 独立解码已经异常 | 查编码输入 YUV、JPEG 编码、输出有效长度及错误/reset |
| P1 正常，P2 字节不同且没有预期转换 | 查复制、同步、并发写入、错误 buffer/offset/长度 |
| P1/P2 一致且异常 | 查上游；必要时保存同一请求的输入 YUV，区分输入图像已撕裂与编码异常 |
| P2 正常，P3 不同 | 查 buffer 发送期间改写、UVC 长度/分片/边界及 host 接收；必要时抓 USB 数据定位首次变化 |
| P1/P2/P3 一致但多个解码器呈现异常 | 查 bitstream、上游输入与编码，避免仅归因显示性能 |
| 原路径坏、复制路径好，但无数据关联 | 只能报告改善，补生命周期证据后再归因 |

## 5. 最小生命周期记录

采用固定大小内存环形日志，异常或测试结束时导出。每个 buffer 的标识应包含分配/复用代次；仅用 fd 或指针可能因复用导致误判。

统一记录字段：

```text
device_monotonic_timestamp_ns
stream_generation / format / frame_interval
request_id / frame_id
source_buffer_id + generation
send_buffer_id + generation
JPEG valid size / capacity / bytesused / offset
事件、状态、返回码
fence 标识及 wait/signal 状态
当前线程 ID / CPU（辅助信息）
```

事件至少覆盖：JPEG_START、JPEG_DONE、FENCE_READY、COPY_START、COPY_DONE、UVC_SUBMIT、UVC_RELEASE、BUFFER_REUSE、DROP、FLUSH、ERROR。

对原共享路径验证：

```text
JPEG_DONE / FENCE_READY 必须先于 UVC_SUBMIT
UVC_RELEASE 必须先于同一发送 buffer 的下一次写入或回收
同一 buffer 同时不能服务两个未释放的帧
提交的 frame ID、buffer、长度必须来自同一请求
```

为每个 buffer 记录最后写入者和所有者；对非法状态转换使用低频告警和事件快照。

## 6. JPEG reset 取证（并行开展，不预设故障）

在 cam_jpeg_enc_reset_hw 调用点/调用方区分原因：初始化、正常清理、stop/flush、电源恢复、timeout、IRQ error、其他。记录 reason、request/frame ID、实例、返回码、done/fence 状态及输出 buffer。

复用驱动已经读取的 IRQ/error status，并记录超时阈值和实际等待时间。不要盲读可能读后清除或具有副作用的寄存器。若一个状态不可取得，应明确报告未知，不用推测值填充。

回答：正常帧是否也 reset？异常 reset 是否先于坏帧？坏帧是否发生在 reset 后首帧？reset 次数应按正常/异常原因分别统计。JPEG IRQ 总次数不是 frame_done 或 reset 次数。

## 7. UVC 源码核查

若 P2 正常而 P3 异常，优先逐项检查：

- bytesused 是否为当前帧实际长度；分片长度之和（扣除协议头）是否等于 bytesused。
- 分片源地址和 offset 是否连续，是否越界、漏片、重复片或使用上一帧残留长度。
- 同一帧 FID 是否保持一致，切帧时变化是否正确；EOF 是否对应当前帧结束，错误标志是否正确处理。
- request completion 是否仅释放自己的引用；整帧还有 request 引用时不能回收帧 buffer。
- short/error/cancel request 是否被统计并正确清理；flush/stop 与完成回调是否存在竞态。
- 在设备发送端与 host 接收端必要时记录分片或 USB 抓包，先确认双方边界，再区分发送、传输及组帧问题。

## 8. 对照测试与验收

固定 Windows host、USB线/端口、3840×2160 MJPEG 30fps、同一场景及 CPU/interconnect 条件。使用持续运动的物体。记录实际协商模式、固件版本、boot ID、运行配置；重新开机后动态获取 PID/IRQ，不能沿用旧编号。

先完成 A/B/A：原路径 → 独立发送 buffer → 原路径。每组建议至少5分钟，至少重复3轮；若基线故障太少，应延长并报告总时长/帧数，不能用一分钟零故障判定修复。

两条测试路径：PotPlayer实时观察用于确认用户现象；FFmpeg原始MJPEG采集用于离线取证。两者尽量顺序执行并保持协商一致；不要默认两个应用共享设备或时序等价。

每组提交：总帧数、实际FPS、可见坏帧数/时间、MJPEG decoder warning/error数、原始坏帧、长度分布、帧间隔异常、buffer非法状态数、复制/等待开销、丢帧数、正常/异常reset数以及CPU/温度/频率统计。

通过条件：

- 在可复现的基线上证明改善，且实际30fps未明显退化、未通过丢弃坏帧掩盖问题。
- 生命周期检查无提前回收、重复所有权或越界；所有异常处理路径同样满足。
- 用同一帧的数据或事件确定异常首次出现的位置。
- 未出现故障需注明测试时长/帧数与基线频率，结论应为测试范围内未复现。

## 9. 开发人员回交内容

1. 实际数据路径说明：Camera输入→JPEG→UMD→UVC；buffer分配器、共享方式、fence/缓存同步及释放语义。
2. 调试开关、修改位置和变更摘要（无需向外公开源码）。
3. A/B/A结果表及原始日志，标注每轮开始/结束的设备时间。
4. P1/P2/P3同一帧对比，至少包含一张故障帧及邻近正常帧；若未拿到，说明原因。
5. reset原因统计和与坏帧的关联结果；无法判断项明确列出。
6. 结论与下一步：确认的事实、待验证假设、是否有性能/稳定性代价。

## 10. 现有证据文件

位于 capture_20261013_115204 目录：

- capture.mkv：两分钟原始MJPEG录像，3350234547字节。
- suspect_96.512.jpg / suspect_103.744.jpg：两张原始异常帧，优先交付。
- suspect_96.512.png / suspect_103.744.png：软件解码结果，便于直接查看。
- verify_96.512.txt / verify_103.744.txt：独立JPEG解码警告。
- packets.json、packet_summary.json：包时间戳、长度、SHA256和统计。
- suspect_decode_log.txt：异常帧定位日志。

另有 UVC_MJPEG_Flicker_Debug_Summary.md（早期背景）和 UVC_Perf_AB_Test_20261013.md（调度实验记录）。JPEG软件解码警告不是JPEG硬件reset证据。

工具说明参考：https://ffmpeg.org/ffmpeg-devices.html#dshow 和 https://ffmpeg.org/ffmpeg.html#Streamcopy 。本交付文档中的代码事件名和状态名为建议，需映射到项目实际实现。
