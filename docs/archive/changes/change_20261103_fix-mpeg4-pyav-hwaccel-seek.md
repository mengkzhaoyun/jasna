# 修复 MPEG-4 Part 2 视频流 VALI 寻址时间戳重置与多卡 NVDEC 上下文绑定变更记录

- **负责人**：Mengsk
- **助手**：Gemini（根因排查、代码修复、单测补充与多卡真机热更新验证）
- **日期**：2026-11-03
- **状态**：已完成
- **所属任务**：T11

## 1. 当前依据

- [实施计划](../../plan/2026.10.multi-gpu-smart-pipeline.plan.md)：T11 任务定义与审批记录。
- [视频硬件解码模块](../../../jasna/media/video_decoder.py)：`_CODECS_SKIPPING_VALI` 编码分流与 `_cuda_hwaccel` 多卡 CUDA 上下文绑定实现。
- [多卡调度引擎](../../../jasna/multi_gpu.py)：L1/L2 子进程解码后端环境变量透传实现。

## 2. 变更内容

### 2.1 将 MPEG-4 Part 2（`mpeg4` / `msmpeg4v*`）纳入跳过 VALI 列表并改走 PyAV NVDEC

1. **根因说明**：
   - 封装在 `.avi` 等容器中的 `mpeg4`（MPEG-4 Part 2，如 `DX50` / `XVID`）视频流在经由 `python_vali`（`VALI`）交由 NVIDIA CUVID `mpeg4` 解析器解码时，每次调用 `avformat_seek_file` + `avcodec_flush_buffers()` 后，解析器都会将解码输出的 `frame->pts` 重置为从 `0` 开始递增的帧序号（`0, 1, 2, ...`），与 `vc1` 的解析器缺陷一致。
   - `VPF::FfmpegDecodeFrame_Impl::SeekDecode` 内部的后置跳帧循环（`while (frame->pts < target_pts)`）在等待该归零计数器追平 `target_pts` 时，每次跳转至 `target_pts` 都会额外多解码跳过 `target_pts` 帧（即 `Seek(T)` 实际落点变成 `2 * T`）。
   - 当跳转目标位于视频后半段（`target_pts >= total_frames / 2`）时，包括 Stage 1 多卡分片扫描中负责后半程的 GPU 工作线程以及 Stage 2 中位于后半程的通道 A 无码片段，`SeekDecode` 都会一直解码至文件末尾并返回 `END_OF_STREAM`（产出 0 帧），导致后半程有效马赛克被漏扫且尾部无码片段触发 `Unrestored span ... produced no frames` 异常。
2. **修复措施（`jasna/media/video_decoder.py` & `jasna/multi_gpu.py`）**：
   - 将 `"mpeg4"`、`"msmpeg4v3"`、`"msmpeg4v2"`、`"msmpeg4v1"` 加入 `_CODECS_SKIPPING_VALI`，使 `auto` 解码后端自动跳过 `VALI` 并切换至 `pyav-hw`（FFmpeg 解码器配合 NVDEC 硬件加速），在保持 GPU 硬件解码的同时严格保留容器原始包时间戳。
   - 在 `jasna/multi_gpu.py` 的 `_run_l2_collaborative_track_denoise` 与 `_render_mosaic_span_on_gpu` 中，当视频编码属于 `_CODECS_SKIPPING_VALI` 时，向工作子进程默认注入 `JASNA_DECODE_BACKEND=pyav-hw`。

### 2.2 修复多卡并发线程下 PyAV NVDEC（`current_ctx=1`）设备上下文错位

1. **根因说明**：
   - `_cuda_hwaccel(device)` 通过 `hwaccel.options["current_ctx"] = "1"` 复用当前线程的 PyTorch CUDA 主上下文。当 Stage 1 多卡线程池中的非主线程（如 `cuda:1..7`）首次创建 `_PyAvFrameSource` 且尚未在该线程调用 `torch.cuda.set_device(device)` 时，FFmpeg NVDEC 会绑定到默认的 `cuda:0` 上下文，而后续色彩转换内核与输出张量分配在目标 `cuda:N` 上，引发跨设备非法内存访问（`CUDA error: an illegal memory access was encountered`）。
2. **修复措施（`jasna/media/video_decoder.py`）**：
   - 在 `_cuda_hwaccel(device)` 入口处显式调用 `set_device(device)`，确保当前线程在初始化 FFmpeg CUDA 硬件加速上下文前已绑定至目标 GPU。

## 3. 执行记录

- 2026-11-03，Gemini：复现 `.avi`（`mpeg4`）视频在 `VALI` 下的后半程 Seek 0 帧问题，定位 CUVID `mpeg4` 解析器 flush 后 `pts` 归零导致 `Seek(T)` 倍增至 `2*T` 的根因。
- 2026-11-03，Gemini：更新 `jasna/media/video_decoder.py`、`jasna/multi_gpu.py` 与 `tests/test_video_decoder_backends.py`，并将补丁热更新至 8 卡 RTX 4090 服务器容器镜像执行真机端到端测试。
- 2026-11-03，Gemini：8 卡 RTX 4090 环境两个长时 `.avi`（`mpeg4`）样本（`13510.0s` / `404,894` 帧与 `7858.5s` / `235,519` 帧）均完整通过 Stage 1 ~ Stage 3 并生成最终 `.mp4`（`HEVC/hev1`）成品，完成变更归档与 CI/CD 流水线发布。

## 4. 验收与验证记录

- [x] `tests/test_video_decoder_backends.py` 中 `vc1` / `mpeg4` / `msmpeg4v3` 自动切换 `pyav-hw` 单元测试通过。
- [x] 8 卡 RTX 4090 环境 Stage 1 全片 8 卡并发扫描 `.avi`（`mpeg4`）全程 `0 ~ 100%` 时间轴覆盖验证通过（后半程 `cuda:4..7` 正常检出目标马赛克）。
- [x] 8 卡 RTX 4090 环境两个 `.avi`（`mpeg4`）任务全流程（Stage 1 ~ Stage 3）真机测试完成并生成最终 `.mp4` 成品（总帧数与时长 `100%` 对齐）。

## 5. 停点与恢复条件

- 停点：无（已完成真机测试、验收与归档）。
- 恢复条件：无。

