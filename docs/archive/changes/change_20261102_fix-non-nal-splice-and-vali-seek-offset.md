# 修复非 NAL 编码切片与非零起始 PTS 视频 VALI 寻址偏移变更记录

- **负责人**：Mengsk
- **助手**：Gemini（根因排查、代码修复、单测补充与多卡环境验证）
- **日期**：2026-11-02
- **状态**：已完成
- **所属任务**：T10

## 1. 当前依据

- [实施计划](../../plan/2026.10.multi-gpu-smart-pipeline.plan.md)：T10 任务定义与审批记录。
- [关键帧切分与混流模块](../../../jasna/media/splice.py)：非 H.264/HEVC 编码随机访问与访问单元边界切分守卫实现。
- [视频硬件解码模块](../../../jasna/media/video_decoder.py)：`_ValiFrameSource.frames` 对非零 `start_pts` 视频流的寻址偏移与前导帧过滤实现。
- [多卡调度引擎](../../../jasna/multi_gpu.py)：`_encode_unrestored_span` 无码片段空帧输出防御校验实现。

## 2. 变更内容

### 2.1 修复非 H.264/HEVC 编码（如 `mpeg4`）切分时的 `KeyError`

1. **访问单元切分守卫（`jasna/media/splice.py`）**：
   - `_next_access_unit_offset()` 原先直接按 `_VCL_NAL_TYPES[codec]` 索引（仅含 `h264` 与 `hevc`），当处理封装在 `.avi` 等容器内的 `mpeg4`（MPEG-4 Part 2 / `DX50`）或 `vp9` 视频流时抛出 `KeyError: 'mpeg4'`。
   - 现增加 `if codec not in _VCL_NAL_TYPES: return len(data)` 守卫，非 H.264/HEVC 编码直接返回整包长度。
2. **随机访问关键帧判定扩展（`jasna/media/splice.py`）**：
   - `_is_safe_random_access_packet()` 由原先仅豁免 `codec == "av1"` 扩展为 `if codec not in {"h264", "hevc"}: return True`，避免对非 H.264/HEVC 关键帧误按 NAL 单元头解析。

### 2.2 修复非零 `start_pts` 视频流在 VALI 硬件解码下的双重偏移

1. **根因说明**：
   - `python_vali` 底层 `VPF::FfmpegDecodeFrame_Impl::SeekDecode` 在将 `SeekContext.seek_ts` 转换为流时间戳并调用 `avformat_seek_file` 时，内部已自动加上一次 `stream->start_time`。
   - `_ValiFrameSource.frames()` 原先在 Python 侧又手动叠加了一次 `start_seconds = float(self.metadata.start_pts * self.metadata.time_base)`，导致当视频流存在较大非零起始时间戳（如 `start_time > 100s`）时，每次 VALI Seek 均向后多跳一倍 `start_time`，使时长短于 `start_time` 的通道 A 无码片段首帧直接越过 `end_pts` 而产出 0 帧，未生成 `.nut` 分片并在 Stage 3 `normalize_fragment()` 报错。
   - 同时，`SeekDecode` 内部后置跳帧循环在 `stream->start_time > 0` 时对 `frame->pts` 重复叠加了 `stream->start_time`，导致 Seek 停留在目标前的关键帧上。
2. **修复措施（`jasna/media/video_decoder.py` & `jasna/multi_gpu.py`）**：
   - `_ValiFrameSource.frames()` 直接向 `SeekContext` 传入相对流起点的 `seek_ts`，不再重复叠加 `start_seconds`；同时计算 `min_pts = self.metadata.start_pts + round(seek_ts / self.metadata.time_base)`，在解码循环中跳过 `frame_pts < min_pts` 的关键帧前导帧。
   - `_encode_unrestored_span()` 在未取消且未生成 `partial_nut` 时直接抛出明确 `RuntimeError`，防止空分片静默流入 Stage 3。

## 3. 执行记录

- 2026-11-02，Gemini：完成多卡服务器批量运行日志与残留断点目录根因排查，反编译定位 `python_vali` `SeekDecode` 时间戳偏移行为。
- 2026-11-02，Gemini：修改 `jasna/media/splice.py`、`jasna/media/video_decoder.py` 与 `jasna/multi_gpu.py`，并在 `tests/test_splice.py`、`tests/test_video_decoder_backends.py` 中补充对应回归单元测试。
- 2026-11-02，Gemini：在多卡容器环境完成真机验证、镜像热更新与损坏断点缓存清理。

## 4. 验收与验证记录

- [x] `mpeg4` / `vp9` / `av1` 非 NAL 编码切分与 `_extract_span_mp4` 实测通过。
- [x] 非零 `start_pts` 视频流在 `_ValiFrameSource.frames` 下的寻址首帧 PTS 与 PyAV 完全一致。
- [x] 目标多卡服务器镜像热更新与残留 `.jasna-resume` 目录清理完成。

## 5. 停点与恢复条件

- 停点：无（已完成全部修复、验证与归档）。
- 恢复条件：无。
