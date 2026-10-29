# EDGS 实验进度与监督记录

研究方向：Reliability-Aware Dense Initialization for 3D Gaussian Splatting。

本仓库记录实验状态、测试指标、证据来源、异常与待确认事项。服务器端由执行窗口负责训练，监督窗口只读核验并同步文档。

## 当前状态

截至北京时间 **2026-10-29 08:33** 的服务器只读核验，当前计划内 **14 组正式实验已完成**：Garden、Bicycle、Bonsai 各四方法，共 12 组；Counter 的 Baseline 与 Unified v2，共 2 组。当时没有训练进程，GPU 空闲。这是已核验的时间点快照。

Counter 在 seed=228、ITER 29228、Evaluating test 下的正式结果：

| 方法 | PSNR ↑ | SSIM ↑ | LPIPS_splat ↓ | Gaussian 数量 |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 29.265922 | 0.922736 | 0.162812 | 572156 |
| Unified v2 | 29.189787 | 0.922883 | 0.162532 | 580180 |
| v2 − Baseline | **−0.076135** | **+0.000147** | **−0.000280** | +8024 |

**Counter 呈指标权衡，不能称为三项全面改善。** 最后 15 个共同测试点中，v2 的 PSNR 胜出 0/15，SSIM、LPIPS_splat 均胜出 15/15。这些点属于同一次训练的相关观测，不能用于宣称统计显著。

**评估口径待确认：** Counter 的原实现将图像限制到 [0,1] 后直接调用默认 `normalize=False` 的 VGG-LPIPS。这里保留原日志字段 `LPIPS_splat`；论文标准 LPIPS 的输入归一化口径需要科研总控确认。两组采用相同实现，本次未修改或重新评估。

## 记录入口

- [当前进度与四场景完成情况](PROGRESS.md)
- [Counter seed228 最终监督报告](reports/counter-seed228-final.md)
- [Counter 正式结果 CSV](data/counter-seed228-final.csv)
- [Counter 后期 15 个对应测试点 CSV](data/counter-seed228-late15.csv)

## 原始日志与模型位置

以下是 `edgs-v1` 容器内路径，用于定位证据。原始模型和完整日志仍保留在实验环境中。

- Baseline 日志：`/workspace/outputs/counter_seed228_baseline.log`
- Unified v2 日志：`/workspace/outputs/counter_seed228_unified_v2_relconf.log`
- Baseline 模型目录：`/workspace/outputs/edgs_counter_seed228_baseline`
- Unified v2 模型目录：`/workspace/outputs/edgs_counter_seed228_unified_v2_relconf`

## 后续更新规则

每次记录注明核验时间、实验名称、seed、已确认的 iteration、明确标记为 test 的指标、Gaussian 数量、日志路径及异常。训练完成需联合核对 ITER 29228 test、ITER 30000 保存、`Training complete.`、最终 PLY 和 checkpoint 文件。

区分原始日志实读、运行记录、计算得到的差值与用户交接数据。Garden 的部分原始日志尚未独立定位，不将交接数值标成已核验。保留 Counter 的 PSNR 下降和 Bonsai 上 v1 优于 v2 的结果。

当前所有结果均为单 seed；渲染指标与 confidence/parallax 诊断不能直接证明三维几何精度改善。
