# EDGS 实验进度

整理日期：2026-10-29（Asia/Shanghai）。

最后一次服务器只读核验：**2026-10-29 08:33**。状态：计划内 14 组已完成，无训练 Python 进程；GPU 利用率 0%，显存 95 / 24576 MiB。此处记录该核验时刻，不代表之后的实时状态。

## 正式实验覆盖

| 场景 | Baseline | SoftGeo | Unified v1 | Unified v2 | 本次证据范围 |
| --- | --- | --- | --- | --- | --- |
| Garden | 已完成（交接） | 已完成（交接） | 已完成 | 已完成 | v1/v2 的 ITER 29228 日志已核验；Baseline/SoftGeo 原指标日志未定位 |
| Bicycle | 已完成 | 已完成 | 已完成 | 已完成 | 四份 ITER 29228 test 原日志与交接数据一致 |
| Bonsai | 已完成 | 已完成 | 已完成 | 已完成 | 四份 ITER 29228 test 原日志与交接数据一致 |
| Counter | 已完成 | 未安排 | 未安排 | 已完成 | 两组完成记录、固定配置、相机划分和最终文件均已核验 |

合计 14 组，不表示四场景四方法的 16 组实验全部完成。

## Counter 结果

统一 seed=228、30000 次迭代、`train.no_densify=True`、`matches_per_ref=10000`、`nns_per_ref=3`、`num_refs=80`。正式比较位置为 ITER 29228 的 `Evaluating test`。

| 指标 | Baseline | Unified v2 | v2 − Baseline |
| --- | ---: | ---: | ---: |
| PSNR | 29.265922 | 29.189787 | −0.076135 |
| SSIM | 0.922736 | 0.922883 | +0.000147 |
| LPIPS_splat | 0.162812 | 0.162532 | −0.000280 |
| 最终 Gaussian | 572156 | 580180 | +8024 |
| 整次运行耗时（秒） | 3593.138804 | 4045.084903 | — |

两组均以退出码 0 结束；ITER 3000、7000、15000、30000 的模型与 checkpoint 均存在。最终 checkpoint 的 CRC 校验通过，最终 PLY 声明的顶点数与文件数据长度一致。

两个日志字段 `N_splats_at_init=155767`、`N_splats_after_init=955767` 已实读。配合 `add_SfM_init=False` 和移除原 SfM 点的源代码，进入训练的初始稠密 Gaussian 数量为 800000。

## 研究结论与待确认事项

1. Counter 上 v2 的 PSNR 降低，SSIM 与原日志 LPIPS_splat 改善，后期 15 个共同测试点均保持这一方向。
2. Bonsai 上 v1 三项指标均优于 v2；不能声称 v2 全面优于 v1。
3. Counter 的 LPIPS 输入归一化口径待科研总控确认：当前实现向默认 `normalize=False` 的 VGG-LPIPS 传入 [0,1] 图像。原始数值保留为 `LPIPS_splat`，未擅自重算。
4. Garden Baseline、SoftGeo 的原指标日志待定位；此前交接表继续作为交接来源，不等同于本窗口独立日志核验。
5. 单 seed 和同次训练的多个评估点不能支持统计显著性结论。渲染指标改善不能直接证明几何精度改善。

下一阶段实验由科研总控决定。本记录未授权启动训练、切换代码或修改参数。

## 容器内日志路径

- Baseline：`/workspace/outputs/counter_seed228_baseline.log` → [完整原始日志](workspace/outputs/counter_seed228_baseline.log)
- Unified v2：`/workspace/outputs/counter_seed228_unified_v2_relconf.log` → [完整原始日志](workspace/outputs/counter_seed228_unified_v2_relconf.log)

日志同步核验时间：**2026-10-29 09:05:20（Asia/Shanghai）**。两份仓库副本均与 `edgs-v1` 中源文件的字节数、SHA256 一致，未经截断、换行转换或内容清洗。此时间是日志同步核验时间，训练进度与 GPU 状态仍对应上文注明的 08:33 核验快照。

| 日志 | 字节数 | SHA256 |
| --- | ---: | --- |
| Baseline | 65905 | `86c01f2e2dea15bccb6b7816b4236d2514eb2168abc6bd47a04dff04f2330d21` |
| Unified v2 | 99045 | `9464a62b63a07b7b1b94a980f64dac8df5e962a1d0288d88cbda37fae95a7d32` |

完整路径映射和源文件修改时间见 [日志同步核验记录](data/log-sync-manifest.json)。之后每次进度检查均同时核验、同步对应日志。

## 进度时间线

| 北京时间 | 已核验进展 |
| --- | --- |
| 2026-10-28 16:38:16 | Counter Baseline 开始运行（原始 run.json） |
| 2026-10-28 17:38:09 | Baseline 完成并正常退出 |
| 2026-10-28 17:38:11 | Unified v2 开始运行 |
| 2026-10-28 18:45:36 | Unified v2 完成并正常退出 |
| 2026-10-28 完成后核验 | 最后 15 个共同 test 点、配置、相机划分、最终 PLY 与 checkpoint 通过检查 |
| 2026-10-29 08:33 | 两组完成证据仍存在；无训练进程，GPU 空闲 |
