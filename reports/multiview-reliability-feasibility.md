# EDGS 多视图几何可靠性最小侵入式可行性报告

核查日期：2026-10-29（Asia/Shanghai）。研究方向：**Coverage-Preserving Multi-View Reliability-Guided Initialization**。

**结论：有条件推荐这一研究方向，暂不推荐直接实现、替换冻结方案或启动正式实验。** 在固定三角化候选上计算额外视角的几何支持，在数据流和计算规模上可行；但当前没有逐候选缓存，投影约定存在必须澄清的源码疑点，也没有独立观测证明新选择更准确。论文提供动机与已有判据，不能证明本报告提出的 EDGS 组合有效。

本轮仅完成文件读取、静态代码/公式审查、既有文件清点及方案设计。**未修改或切换 `corr_init.py`，未改任何冻结快照，未执行 RoMa、三角化数值试验、训练、重评或新数据导出，未删除、移动模型或日志。** GitHub 仅新增本报告，不改既有结果与进度记录。

证据标签：**[核实]** 指实际服务器文件或指定官方论文/代码；**[推导]** 指由已核实公式得到的分析；**[设计]** 指尚未实施的建议；**[缺证]** 指目前无法确认或验证。

## 核查范围与版本

- **[核实]** 重新登录后的实际主机为 `3090-server`，运行容器为 `edgs-v1`，挂载为宿主 `/home/ttj/workspace` → 容器 `/workspace`。项目 `/workspace/projects/EDGS-main`、数据 `/workspace/datasets/mipnerf360`、输出 `/workspace/outputs` 均存在。本轮初检时间为北京时间 09:39:07；无训练进程，RTX 3090 显示 0% 利用率、约 95 MiB 显存。
- **[核实]** 当前生效文件与冻结 v2 相等。实际文件 SHA256 如下，不能把公开仓库 HEAD 当作这份实验源码的版本号。
- **[核实]** 北京时间 10:01:12 结束只读检查时，生效文件与四个冻结快照的 SHA256 均与初检相等；容器中仍无训练进程。没有为本报告创建服务器端算法或诊断程序。

| 实际文件（相对项目） | SHA256 |
|---|---|
| `source/corr_init.py` | `5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88` |
| `source/corr_init_baseline_seed228.py` | `9f3199443473f127c349b926fc3e7d6ff209f17a8cb8854fd8a5cb8be8dc3c10` |
| `source/corr_init_softgeo_l1_tau2.py` | `4257d209a4fdbad850c39d69e76f5ad8d44887fc1dc9b81675c7c7a2924ba17b` |
| `source/corr_init_unified_v1_frozen.py` | `63a90f0b5bcd784d288e58bbc06746b9e2df78f2fb056c5fc4815f1de6d1b410` |
| `source/corr_init_unified_v2_relative_conf.py` | `5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88` |

- **[核实]** 既有记录为 14 组：Garden/Bicycle/Bonsai 各四方法，Counter 两方法，并非四场景完整 16 组。Counter 的 PSNR 下降、Bonsai 的 v1 优势与 LPIPS_splat 口径待确认事项，均沿用[现有记录](../PROGRESS.md)，本轮没有改值或重算。渲染质量与 confidence 统计不能直接证明三维几何准确性。
- 官方 EDGS 固定为 [`f90b022445fc88368f75e66e8fb34aea88372cac`](https://github.com/CompVis/EDGS/tree/f90b022445fc88368f75e66e8fb34aea88372cac)；TVG-SLAM 固定为 [`6f84b0b94b198b93e39e6497920814c662408374`](https://github.com/MagicTZ/TVG-SLAM/tree/6f84b0b94b198b93e39e6497920814c662408374)；VGGSfM 固定为 [`e1d9d2eb2b3575525792206fb94b2c749c58dc50`](https://github.com/facebookresearch/vggsfm/tree/e1d9d2eb2b3575525792206fb94b2c749c58dc50)。以下公开代码链接均固定 commit，实际 v2 行号另行标注。

## A. 原始 EDGS 的局限与源码证据

### A1. 原选择确实只验证候选的两个生成视图

**[核实]** 官方 [`triangulate_points()` L412–488](https://github.com/CompVis/EDGS/blob/f90b022445fc88368f75e66e8fb34aea88372cac/source/corr_init.py#L412-L488) 对参考帧与一个邻居的匹配解批量线性最小二乘，返回该候选及两个投影误差。误差实际为 NDC 中两个分量的 **L1** 绝对差之和。官方 [`select_best_keypoints()` L492–517](https://github.com/CompVis/EDGS/blob/f90b022445fc88368f75e66e8fb34aea88372cac/source/corr_init.py#L492-L517) 使用

\[
E_i=\max(e_{r,i},e_{i,i}),\qquad i_0=\arg\min_i E_i.
\]

这里没有把同一个候选投影到其余邻居来检验。小生成视图误差不排除小基线退化、错误匹配或共同偏差；这是几何上的动机，**不是本轮已测得的误选率**。实际 v2 的 L506–580 在同一误差上增加既有视差和相对 confidence 乘子，仍没有 out-of-pair 观测残差。

### A2. EDGS 已有的内容不能再次包装为创新

原始代码已经包括 RoMa confidence、跨邻居逐像素 confidence-max 聚合、confidence 引导采样、批量三角化、双视图重投影选择及高误差 opacity 抑制。见官方 [L100–287](https://github.com/CompVis/EDGS/blob/f90b022445fc88368f75e66e8fb34aea88372cac/source/corr_init.py#L100-L287)、[L597–684](https://github.com/CompVis/EDGS/blob/f90b022445fc88368f75e66e8fb34aea88372cac/source/corr_init.py#L597-L684)。`certainty > sample_thresh` 是饱和到 1，不是把其余低 confidence 全部剔除；不能将该阈值直接改称可见性门限。

**[核实]** 修订论文 [EDGS arXiv:2504.13204v2，§3.3–3.4，Eq.(4)–(11)](https://arxiv.org/html/2504.13204v2#S3.SS3) 已讨论三角化、重投影及 confidence/几何联合采样和跨邻居聚合；Table 6 是采样组合的渲染质量消融。论文 Eq.(8) 的 L2 表述与代码 L1、论文采样分布与当前公开代码的具体流程需分开，不能互相替代。**跨邻居聚合采样不是候选的多视图一致性验证，但它本身不是我们的新贡献。**

### A3. 投影约定有真实源码疑点，尚未测定影响

实际 `triangulate_points()` L441–462 用 `full_proj_transform` 的 0-based 列 0、1、**2** 建方程；L479–486 却除以投影结果的分量 **3** 加 EPS 来算误差。实际相机实现与官方固定 GS 子模块一致：`world_view_transform=W2C.T`，`full_proj_transform=world_view_transform @ perspective.T`。由 [camera L80–89](https://github.com/graphdeco-inria/gaussian-splatting/blob/54c035f7834b564019656c3e3fcc3646292f727d/scene/cameras.py#L80-L89)、[projection L51–70](https://github.com/graphdeco-inria/gaussian-splatting/blob/54c035f7834b564019656c3e3fcc3646292f727d/utils/graphics_utils.py#L51-L70) 可推导

\[
\mathrm{clip}_w=z_{cam},\qquad
\mathrm{clip}_z=\frac{f}{f-n}z_{cam}-\frac{fn}{f-n},\quad n=0.01,\ f=100.
\]

两列并不等价。这是**静态源码的分母约定不一致**，不是已执行数值验证的 bug 结论，更不能直接解释已有 14 组结果。新方案接入前必须检查坐标闭环；本轮不修正旧三角化、不重算旧结果。旧 pair error 要保留用于公平回退和 opacity 合约，新的物理投影诊断应单列，不能暗中替换旧误差定义。

## B. 论文与官方实现对照

| 年份与工作 | 真正任务及可靠性机制 | 已核实实验及证据边界 | 官方代码与适配程度 |
|---|---|---|---|
| 2025，TVG-SLAM；arXiv v1，仓库声明 RA-L 接收 | RGB-only 增量 GS-SLAM；DUSt3R 三视图 track、tracking 几何损失，TUGI 按多份 3D 估计散度调 Gaussian scale/opacity | Table IV 单个 Waymo 153495 场景：完整 ATE 0.870 m / PSNR 25.62，去 TUGI 为 1.203 / 24.90。只支持其系统中组合模块的价值，未隔离散度、scale、opacity，也未验证 EDGS 固定三候选排序 | [官方仓库固定目录](https://github.com/MagicTZ/TVG-SLAM/tree/6f84b0b94b198b93e39e6497920814c662408374)仅 README/assets，无核心源码；不能直接复制模块 |
| 2024，VGGSfM；论文全文核验 arXiv v1 | 无序图像联合恢复相机和稀疏结构；论文 learned triangulator、可见性掩码 BA、组合过滤；公开 main 使用 RANSAC DLT、多视图角残差、支持数、LO/BA | 论文 Table 3 固定相机 ETH3D：Accuracy@5cm 96.52%、Completeness 33.96%；Table 6 无组合过滤 AUC@10 为 2.31，完整为 73.92。并非单个角残差/阈值的因果证明 | [官方实现](https://github.com/facebookresearch/vggsfm/tree/e1d9d2eb2b3575525792206fb94b2c749c58dc50)可读；评分思想与小函数最直接相关。不能整体接入 LO/BA 或换掉 EDGS 三角化 |
| 2026，EDGS；arXiv 初稿 2025、修订 v2 2026 | 已知相机下以稠密匹配初始化减少 densification 需求；本身已有成对几何误差和 confidence 机制 | Table 6、Gaussian 轨迹分析支持原方法，不证明本报告的新可靠性模块。已有本项目渲染指标也不构成 3D 真值精度证据 | [官方代码](https://github.com/CompVis/EDGS/tree/f90b022445fc88368f75e66e8fb34aea88372cac)与实际冻结分支分开核验；保留匹配器、三角化核和训练主体 |

论文原文：[TVG-SLAM arXiv:2506.23207v1](https://arxiv.org/html/2506.23207v1)，[VGGSfM arXiv:2312.04563v1](https://arxiv.org/html/2312.04563v1)及[CVPR 2024 官方条目](https://openaccess.thecvf.com/content/CVPR2024/html/Wang_VGGSfM_Visual_Geometry_Grounded_Deep_Structure_From_Motion_CVPR_2024_paper.html)，[EDGS v2](https://arxiv.org/html/2504.13204v2)。TVG 的 IEEE 最终全文未成功读取；不声称期刊最终版公式与 arXiv 完全相同。

VGGSfM 版本尤其重要：README 建议 v1.1 复现，但 [v1.1 triangulator L45–52](https://github.com/facebookresearch/vggsfm/blob/7dc711d05d2db3be5847d02d1aba06fccea16a8d/vggsfm/models/triangulator.py#L45-L52)已移除可学习参数；[main L30–40](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/models/triangulator.py#L30-L40)也说明这一变化。不能用论文 learned 模块的实验来背书 main 的每个阈值。

## C. 数学依据与推荐度量

### C1. 已有公式：TVG 的散度不是候选排序分数

TVG §III-E1、PDF p.4、Eq.(8) 对共同坐标系内同一 track 的估计定义

\[
\sigma_g^2=\frac1N\sum_i\|P_i-\bar P\|^2.
\]

TUGI 随后调 scale/opacity，不是给 EDGS 的每个候选一个 `argmin` 分数。[原文 §III-E](https://arxiv.org/html/2506.23207v1#S3.SS5)。**[推导]** 若同组所有候选共用一个正乘子 `f(sigma_g²)`，则 `argmin S_i f(sigma_g²)=argmin S_i`，不会改变选择。改成到均值的距离虽可排序，却只反映候选间接近程度，不能排除共同偏差；原始米制散度也受深度和场景尺度影响。

TVG Eq.(4)–(5) 的点/线记号另有歧义：原文把转移量称 epipolar line，但展开式是叉积型残差；如果真为直线系数，点线 incidence 应为内积。没有核心代码可澄清，因此**不推荐照搬这一 trifocal tracking loss**。该疑问不等于已证实作者实现错误。

### C2. 已有公式：多视图投影与角残差

VGGSfM 论文 §3.4 Eq.(6) 是带 visibility 掩码的多视图 BA 目标。本报告只借鉴观测残差，不执行 BA。其公开 [`calculate_normalized_angular_error_batched()` L431–472](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation_helpers.py#L431-L472) 可等价写为

\[
\hat b_j=\frac{K_j^{-1}\tilde u_j}{\|K_j^{-1}\tilde u_j\|},\quad
\hat d_j(X_i)=\frac{R_jX_i+t_j}{\|R_jX_i+t_j\|},\quad
a_{ij}=\arccos\!\left(\operatorname{clamp}(\hat b_j^T\hat d_j(X_i),-1,1)\right).
\]

这是**源码公式的等价表述**，不是 TVG 或 VGGSfM 论文新增编号公式。函数实际输入的 2D 已是去标定相机平面坐标；不能重复乘逆 K，也不能把 EDGS 的 NDC 当作该坐标直接传入。

**[设计：按 EDGS 相机约定适配]** 令实际 row-vector 矩阵为 `W_j`、`F_j=full_proj_transform`，`W_j.T` 的前三行定义传统 `[R_j|t_j]`。对未改变的候选 `X_i`：

\[
h_{ij}=[X_i^T,1]F_j,\quad z_{ij}=([X_i^T,1]W_j)_z,\quad
\pi_j^{ndc}(X_i)=h_{ij,xy}/h_{ij,w}.
\]

先检查有限性和深度，再计算残差；不把 NaN 转成 0。像素残差为

\[
e_{ij}^{px}=\left\|\operatorname{diag}(W_j^{img}/2,H_j^{img}/2)
(\pi_j^{ndc}(X_i)-u_j^{ndc})\right\|_2.
\]

上式用 `W_j^img` 区分图像宽度与 world-view 矩阵。对于当前中心主点/FoV renderer，可直接由已规范为 `[x,y]` 的 NDC 构造 bearing：`[u_x*tan(FoVx/2), u_y*tan(FoVy/2), 1]` 后单位化；若用原始 K，必须包含 resize/crop/去畸变变换。新残差与旧 NDC-L1 pair error 分开存放。

### C3. 退化与置信度的证据边界

- 正深度应检查 camera-space z，不是 `clip_z`；投影分母接近零、非有限坐标须显式标记。没有观测时不能据“投到相机后方”推断它本应可见。
- VGGSfM [角度实现 L475–587](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation_helpers.py#L475-L587)使用 `min(theta, pi-theta)`，接近 0°、180°均可能退化。当前 EDGS v2 统计使用未折叠的 acos 角并施加既有软惩罚；本轮不改它，也不能简单宣称“角越大越可靠”。
- RoMa certainty 是匹配网络输出，**不是已标定的可见性概率或像素逆方差**。只能在经验证的 observation gate 或分析分层中使用，不能直接当作协方差权重。
- VGGSfM main 的 angular=2°、min angle=1.5°、visibility>0.05、score>0.5 是[源码默认值](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation.py#L677-L710)；论文附录另有 3°、3 px、visibility 0.6 等组合，**不直接移植为 EDGS 新阈值**。EDGS 既有 SoftGeo 的 tau=2°也不是新硬退化门限。
- **[设计]** 尚无本项目可直接确定的新阈值。角残差门限应由仅训练数据的开发集观测噪声/分辨率校准；小角近似下可用 `delta_angle ≈ delta_pixel/focal` 联系单位，仍须验证假设。任何 confidence、角度、支持数门限均需来源记录和后续消融，不在本报告直接定值。

### C4. 可单独参照的官方函数

| 函数/机制 | 可取部分 | 不能直接照搬的部分 |
|---|---|---|
| VGGSfM `calculate_normalized_angular_error_batched` | 对固定候选批量算角残差 | OpenCV `[R|t]` 与相机平面坐标输入需适配 |
| `project_3D_points`、`cam_from_img` | 投影/去标定公式 | `img_from_cam` 有 `nan_to_num`；不能让无效投影变成好匹配；也不能传 EDGS 4×4 full-proj 当 K |
| `calculate_triangulation_angle_*` | 相机中心、折叠角诊断 | 场景尺度与阈值不能直接沿用 |
| `calculate_residual_indicator` | 支持数优先、残差决胜的思想 | 不照抄跨 batch 最大值归一化作为新经验权重 |
| `triangulate_tracks_single_chunk` | 全 track 支持判据的参考 | 函数内 LO 会重三角化；其上层管线另有轨迹过滤和 BA 修改相机/点，均不接入本方案 |

参考：[支持数/残差 indicator](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/two_view_geo/utils.py#L63-L87)、[候选与 LO 流程](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation.py#L776-L956)、[投影 helpers](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation_helpers.py#L310-L429)。直接 import 整个文件会引入额外 SfM 依赖；其 [CC BY-NC 4.0 许可证](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/LICENSE.txt)也须在后续实际复用前遵守。本轮只做数学/接口分析。

## D. 当前 EDGS 数据结构逐项检查

以下实际行号对应开头 SHA256 的 v2 `source/corr_init.py`。

| 所需数据 | 运行时是否有 | 当前保存产物是否足够 | 源码/实读证据 |
|---|---|---|---|
| 同一参考像素 X1/X2/X3 | 有，`[3,M,4]`，w=1 | 未找到逐候选缓存 | L783–829 三个 NN 各三角化一次；L834 stack |
| 各 NN 的二维匹配与 confidence | 有，`warps_all`、`certainties_all`及 sampled 局部变量 | 无对应的密集匹配导出 | L742、769–786；confidence 与 warp 用同一 `good_samples` |
| 两个生成视图的误差 | 有，两个 `[3,M]` | 日志没有逐点数组 | L795–799、822–824、835–836 |
| 视差角与相机位姿 | 有；v2 存 angles，camera 对象有矩阵/中心 | 日志只有逐参考视图统计；模型 camera JSON 不能代替完整运行时变换 | L801–829；实际 camera L80–89 |
| 原三个邻居以外的密集匹配 | 图像/相机可访问；匹配未计算 | 没有可直接使用的第四视角 warp | L700、712、742 只构造选定 NN 的 star edges |
| 参考像素 IDs / camera 映射 | 局部有 `good_samples`、source/NN 索引 | 未持久化与候选对应关系 | L764–786；NN 索引是 split 后 train-list 位置，不是 camera.uid |
| 可见性/遮挡真值 | 未见 | 无 | confidence、FOV、正深度都不是遮挡真值 |

### D1. 索引对齐：有明确构造证据，但不等于正确物理 track

1. L167–173 为每个 NN 手建同一 source grid，前两分量为 **[y,row; x,col]** 的顺序，目标 flow 后两分量为 **[x,y]**。同一个参考帧和相同模型分辨率下，源网格由同一表达式构造。
2. L238–247 将所有 NN 的 warp/confidence resize 到**第一个 NN 图像的 H/W**，`align_corners=False`。L764 只采样一次，L783 和 L786 对每个 NN 使用完全相同的扁平索引。
3. 这比“shape 相同”强：确有共同参考网格和共同 sample IDs。但三个目标匹配仍可能落在不同真实物理点上；遮挡、误匹配和共同偏差未被排除。
4. L315–317 对 RoMa 的 H/W 参数看似交换，需结合 source 的 [y,x]、L392–397 的归一化及最终交换理解，不能单行判错。实际包路径已用不导入模型的模块查找核实为项目的 `submodules/RoMa/romatch`；其 `matcher.py` L716–732 签名是 `(coords,H_A,W_A,H_B,W_B)`。
5. `extract_keypoints_and_colors()` 的 `kptsB_proj_matrices_idx` 来自 confidence-max 所属邻居，主要用于复合颜色读取；真正三角化的 P2 在 L794 明确用当前 `NN_idx`。新模块不能把这两个相机索引混同。
6. **[缺证]** 历史运行时数组已释放，未执行“各 NN 源坐标逐点相等”的数值断言。未来导出需同时保留 actual source coordinates、sample IDs、image尺寸和相机 UID，检验像素中心/resize边界及异尺寸图像情形。

### D2. 坐标、尺寸与 split

实际 RoMa `to_pixel_coordinates()` 使用 `W/2*(x+1), H/2*(y+1)`，不是 `(W-1)/2`。不能混用 edge-centered 坐标、离散像素索引及 OpenCV 的另一套约定。实际 `matcher.py` SHA256 为 `acefa44a0d983be14542239b4dd43d939f1fcdd5b1c0a222dc2b964b9ffb8973`；camera 文件为 `7344b9a4c95e3649becab1b9e30eb0b351a15a61370f64352dcb231aed916128`；graphics 文件为 `d6e75de8161e98b72c2691e8a944a842e1b6625d2306035d079c868ac632edfc`。

Counter 模型的 `cameras.json` 有 240 条，原图尺寸均为 3115×2076；这是 **CameraInfo 原图尺寸**，不是运行时 tensor 尺寸。按实际 loader 的 `resolution=-1`、scale=1 与 int 截断规则，名义推算为 1600×1066；本轮未重新构造 Camera 对象来实测，显存估算也明确以此为假设。源帧与 NN 若异尺寸，L238 的第一邻居网格以及 L392–397 的归一化尤其需要验证。

实际 `source/networks.py` L26–33 在初始化前调用 train/test split，`source/data_utils.py` L13–22 在无既有测试相机时移出每第 8 张，corr_init L700 仅取 `getTrainCameras()`。**新增评分、开发校准及独立可靠性验证都只使用这一划分后的训练列表**；不能只看 `dataset.eval=False` 就认为可以使用全部图像。

### D3. 已保存的中间数据不足以直接做新排序

对项目与 `/workspace/outputs` 做只读文件名清点：没有候选/匹配用途的 `.npz/.npy/.pt/.pkl/.pickle`，也未定位 match/track/correspondence 缓存。找到的名称相似文件是源码/测试或渲染器资料，不是实验数据。该结论限定于已检查路径，不宣称全服务器不存在别处缓存。

实际 `reference_image_dict` 每 ref 临时创建，返回只有相机列表、邻居索引和 `visualizations`；主入口没有持久化该字典。`GaussianModel.capture()` L68–82 保存 Gaussian 参数、梯度统计与 optimizer 状态，没有三候选、good_samples、RoMa warp 或原始观测。已有 checkpoint 从 ITER 3000 起且 Gaussian 已优化/剪枝，最终 PLY 也不能逆推出未选的候选。

因此：可以利用现有代码、相机文件、原始日志审查协议和成本；**不能声称已在旧产物上离线验证新候选分数**。COLMAP sparse track 可作为另行核查的稀疏几何参照，但不自动与稠密 sampled pixels 对齐，也不是独立稠密真值。

## E. 最小接入位置与接口

**[设计，未实施]** 决策范围可局限于 `source/corr_init.py`：

| 位置 | 最少变动的职责 | 保持不变 |
|---|---|---|
| `init_gaussians_with_corr()` L769–841 | 将 sampled 2D/confidence、sample IDs、camera IDs/矩阵、观测有效性传给 selector；可在未来另行批准后导出 | RoMa 网络/权重、三次原三角化、原采样预算与候选坐标 |
| `select_best_keypoints()` L492–670 | 对原 `[3,M,4]` 候选作可靠性判断与选择；诊断另存 | 输出 point 数量与接口语义 |
| Gaussian 写入 L885–917 | 继续消费原始 pair error 和原颜色/scale/opacity逻辑 | 不让新 score 替代旧误差 |
| `trainer.py`、Gaussian 优化、renderer | 无需接入新决策 | 全部不改 |

当前 selector 输入虽有 X、pair errors、angles、confidence，**没有二维匹配、相机矩阵及采样 ID**。所以“只改变 selector 决策、不动训练主体”可行；“严格只改函数体、调用处和输入都不动”不可行。

实际 v2 返回 `(selected_X, selected_raw_pair_error, selected_nn_indices)`。对新选出的 `i*`，第二项必须是 **`E[i*,m]`**；不能返回新多视图 score，也不能在选择另一个 X 后仍返回 `min_i E_i`。L895–898 按此旧误差设置 opacity，混用会暗改下游逻辑。未来接入应保留签名兼容策略和独立诊断字段，不改冻结文件本身。

## F. 排序、无效回退与覆盖保持

以下是研究设计，不是已经验证的新算法，也不是对当前 v2 公式的修改。

### F1. 分开“观测不存在”和“候选不符合已有观测”

- 每个像素先有候选无关的 observation 状态：已知可用、缺失、遮挡/可信度不明。confidence 高也不能单独证明未遮挡；当前没有反向 cycle 或深度遮挡真值。
- **缺失/未知观测**不补零误差，也不无条件赋无限误差；以相同规则对各方法标 missing，报告有效观测率。
- 若某视角的匹配确认为可用，而候选在该视角负深度、出界或无效投影，这是该候选的几何反证，不能通过删除该残差让它获利。
- 候选在两个生成视图中的有限性、正深度和退化诊断须单列。保守初始设计只在三个邻居观测均可用时重排，其余回退；这避免不同候选拥有不同可用观测分母。是否放宽为一个额外视角支持，需未来消融，未在这里硬定新参数。

这些有效性规则会改变固定候选集中“哪些候选可选”，属于选择资格检查，不能将全部变化笼统称为一个纯连续分数。它们不删除候选数组或减少输出条目，但其影响也需要单独消融。

### F2. 无新融合权重的候选特定支持排序

设 `X_i` 由 `(r,i)` 构造；内部评分只使用 `j≠i` 的其他邻居。**[设计]** 可参照 VGGSfM，记经过观测有效性判断的角残差支持数为

\[
n_i=\sum_{j\in V_{score}\setminus\{i\}}m_j^{obs}\,\mathbf1[a_{ij}\le\tau_{a,j}].
\]

在候选几何有效且证据足够时，采用词典序：**支持数多 → 同支持数下 inlier 角残差均值小 → 原 pair error 小 → 原 NN 索引决胜**。这是一项有既有机制依据的组合设计，**不是从论文直接摘取、已被证实有效的 EDGS 新公式**。阈值 `tau_a` 和 observation gate 尚待仅训练开发集校准；若无校准证据，可以先只分析连续残差与排序相关性，不启动生产式选择。

零 inlier 的均值定义为不可用/正无穷，全部零支持则回退。只在 inlier 中决胜会忽略超阈值残差的具体大小，是这一设计的偏好与待消融点，不能保证所有视角的几何反证都减小。

`acos` 角残差默认弧度，`tau_a` 必须同单位；显示为度时显式转换，不能把源码默认的“2°”当成数值2弧度。

三个 out-of-pair 集合各有两个视角，但不是同一组视角，仍可能产生系统差异；不得把内部更小的评分当成独立验证结果。若有缺失导致支持基数不等，先回退/分层报告，而非只比较少量“容易”残差的均值。

### F3. 回退与 opacity 资格保护

1. 回退对象明确为**冻结 Baseline 的原 pair-error argmin** `i0`，不是不加说明地回退当前 v2。实际 v2 文件保持原样。
2. 新证据不足、支持冲突或无可接受候选时，保留 `X_i0`，记下 fallback 原因，不删点、不复制其他像素点凑数、不平均/重三角化生成新点。
3. 为严格保持旧 opacity 资格，本保守设计**只对 Baseline 的 E_i0 有限且通过原 cfg.proj_err_tolerance 的样本允许重排**，新候选也必须通过同一旧阈值；其他样本一律原回退。于是旧门限的 pass/fail 布尔值不变。Counter 既有阈值为 0.01；这是沿用原合约，不是新增融合权重，后续仍需审核这一保守限制。
4. 若原始候选全部非有限或无正深度，原回退只能保留一个名义条目，**不能同时保证输出几何有效**。必须报告 `invalid_fallback`，交总控决定后续处理，不能暗中修补被冻结实现或宣称覆盖完好。

这一记录要求也适用于仅原 `i0` 的 X/E 非有限或深度无效的情况，不限于全部失效。原 `argmin` 前没有有限性屏障，且 `NaN > threshold` 为 False，不代表有效误差通过检查。合约核对应使用实际旧 `mask_bad=(E>threshold)`，另外单独报告 finite/depth 标志，不能把原 opacity 未被抑制解释为几何有效。

### F4. “Coverage-Preserving”应限制其含义

固定参考帧、相同 `good_samples`、每 sampled pixel 保留一个原候选，可保持**名义点数与参考采样 footprint**。它不能自动保证三维体素覆盖、表面完整性、正深度覆盖、有效 opacity 覆盖或训练后剪枝覆盖。

实际 L895–898 会抑制高误差点，`trainer.prune()` L258–263 仍按 opacity 剪枝；`no_densify=True` 不等于不剪枝。因而研究名可暂用上述方向，但论文必须报告有效覆盖证据，不能只用 800000 个数组条目证明“覆盖保持”。

## G. 三邻居、独立验证与成本：七个关键问题

| 问题 | 明确回答 |
|---|---|
| 1. 三个邻居够不够检查跨视角一致性？ | 够做内部 out-of-pair 检查：每候选由参考+一个邻居生成，可在另外两个邻居检验。当前一共是 4 幅图，不等同于 TVG 的 3 幅总视图。 |
| 2. 全三个邻居用于三候选/评分后还有独立观测吗？ | 没有最终选择之外的验证观测。对某候选“未用于三角化”不等于“未用于最终选择”；也不意味着误差统计独立。 |
| 3. 必须额外一个训练视角吗？ | 若保留全部三个原候选并用尽三个邻居评分，至少需第 4 个训练邻居只做验证，即参考+4邻居共5幅图。另一方案是预留现有一个邻居，但须将其从候选生成及 confidence-max采样排除，候选池会改变，不是原3候选严格对照。若新增视角又用于评分，还需另一个留出或预先分配 folds。 |
| 4. 新视角是否需重跑 RoMa、多少时间？ | 有正确方向/分辨率且 sample IDs 可关联的缓存才可复用；现在未找到。每 ref 新增一个 pair，80 refs 新增80次；不是无匹配的几何投影就能获得观测。额外时间见下文条件估算。 |
| 5. 能否先用旧匹配做离线分析？ | 数据流设计可以，当前保存产物不行：候选、IDs、密集观测未保存。现有日志只能给聚合统计。现在不能宣称旧结果已支持新方案。 |
| 6. 最小导出什么？ | 见 H1：候选、ID/相机映射、二维观测/confidence、旧误差、坐标变换、split/版本、独立验证观测。无需保存所有全分辨率 warp。 |
| 7. 能否只影响 selector、不改训练？ | 能；但要在同一 corr_init.py 调用处补输入。严格只改原函数体不够，且必须维护所选原误差与 opacity 合约。 |

**独立性协议 [设计]：** 第四邻居应在看到其匹配残差前按训练相机和预定规则确定，完全不进入该参考像素的采样 max-confidence、候选生成、评分或阈值校准；只在选择冻结后评价。额外匹配最好在原候选/采样全部固定后单独分析，避免新增调用影响 RNG 状态，不能只凭同 seed 假定候选池不变。这里的独立是“未参与该次选择”，仍共享参考图像、网络和位姿误差，并非噪声统计独立；全局阈值开发集与最终内部验证组还需按相机/track 分组预先隔离。

### G1. 计算复杂度与显存（推导估算，非跑分）

设 R=80、M=10000、K=3。每候选检查另外两个邻居，共 **60000 次 point-view 检验/ref、480万次/scene**；统一算 `[K,3,M]` 后屏蔽对角是每 ref 90000 次。复杂度 `O(R*M*K*V)`，不需要 `M×M` 点对比较；逐 ref/分块、无梯度的 PyTorch 计算在结构上可行。

FP32 下 `[3,3,10000,3]` camera-vector 约 1.08 MB，二维预测约 0.72 MB，标量残差每份约 0.36 MB；mask、深度、归一化与临时量另计。**新评分 tensor 的工程预算可设为每 ref 数 MiB 至约20 MiB**，不是已测得峰值，尚未包含 allocator/workspace 和原匹配器内存。

若假定实际图像网格 1600×1066，单视角 dense warp(4通道)+confidence 的 FP32 数据约34.1 MB；三个约102.3 MB/ref。不要让80 refs全部常驻，三邻居全量缓存约8.19 GB，且没有必要。只保存 sampled bundle 更合适。

### G2. RoMa 调用数与时间预算

- 原始主循环 `80×3=240` 个 pair，加一次独立 warm-up。只在运行时已有三个匹配上做评分，不增加网络 forward。
- 加第四邻居验证时主循环对应 `80×4=320` 个 pair，**匹配对数增加约33.3%，不等于整个初始化时间增加33.3%**。
- 目前旧缓存已丢失；若未来获准从旧实验补建完整 bundle，需要重新获得原240对及新增80对，不能把“现在补数据”的总成本说成仅80对。
- 条件成本为 `DeltaT = 80*t_pair + resize/transfer/export overhead + scoring overhead`。例如**仅作为假设**取 `t_pair=0.25 s`，新增80对为20 s，尚未加其他开销；这不是3090实测。EDGS 论文附录的 A100 约0.21 s/pair也不能当本机保证。
- 既有日志有无阶段标签的 `3/3` 进度条，混有匹配和三角化循环；不能从统一的迭代速率准确分离 RoMa 单耗时。本轮未跑新 profiler，也不承诺“毫秒级/零开销”。未来授权后应独立计时读取、匹配、采样、三角化、评分、导出及 GPU 峰值。

## H. 离线验证设计与最小数据导出

### H1. 最小 bundle（待另行授权导出）

| 字段 | 用途 |
|---|---|
| scene、seed、快照/配置 SHA、软件版本、split清单 | 复核实验与防正式测试集泄漏 |
| ref/NN/validation 的 train-list index、camera.uid、colmap_id、image_name | 防止重排后相机索引错配 |
| `good_samples`、原网格H/W、canonical `[x,y]` source坐标 | 证明三候选和观测确为同一参考采样；固定覆盖与 RNG 控制 |
| `X[3,M,3]`及齐次w约定、两个原pair-error数组 | 不重三角化即可比较候选；维护旧 opacity/fallback 合约 |
| 各视角 sampled 2D匹配/confidence、obs状态 | 跨视角残差；缺测不能当零误差 |
| 实际 image H/W、FoV或变换后的K、W2C/full-proj、resize/crop/去畸变约定 | 正确投影、深度与像素/角单位 |
| 原 Baseline 选择、是否opacity-pass、原NN次序 | 配对与回退审计，可由完整误差重建但建议显式记录 |
| 完全留出的训练视角匹配、ID与可用性 | 独立于选择的检验，不能混进score |

视差角可由 X 与相机中心重建，非必须重复存储；诊断额外可保存 fold-angle、秩/条件数、fallback原因。反向 cycle/遮挡证据若不曾计算，应标 unknown，不能伪造 visibility mask。原始数据以只读副本分析，所有方法使用完全相同 bundle。

**[推导]** 全场80×10000点的基础 FP32 bundle：3×XYZ=28.8 MB、三视角2D=19.2 MB、confidence=9.6 MB、两份pair error=19.2 MB、source2D=6.4 MB、int64 sample IDs=6.4 MB，总计约 **89.6 MB**，未含mask/metadata。新增一个验证视角2D+confidence约9.6 MB；额外保存齐次坐标和parallax会继续增加。无需导出全量图像 warp。

### H2. 必须先通过的坐标与数据闸门

本轮未执行以下数值检查；它们属于后续申请的前置工作：共同sample IDs/源坐标等值；相机UID与split映射；非方形图像和resize边界的坐标闭环；正深度及renderer投影一致性；clipZ/clipW疑点影响；失效/NaN/遮挡记录；旧误差与所选candidate返回值相符。不得以新模块之名顺带修复被冻结三角化。

### H3. 如何判断排序是否“更可靠”

1. **配对、选择前冻结。** 同一参考像素、同三候选、同观测状态，比较原 Baseline、冻结v2与拟议选择；先固定评分/校准协议，再查看独立训练视角残差。正式测试图像不用于候选评分、几何校准或本阶段独立可靠性分析。
2. **独立观测一致性。** 在未参与该次选择的训练视角上比较像素/角残差的均值、中位数、尾部及失败率。报告共同可观测集合和总集合的 missing/fallback率，不能只保留新方法表现好的可见子集。该残差是多视图一致性 proxy，不等于3D真值误差。
3. **几何稳定性。** 未来可在校准的2D噪声/位姿扰动下复用原 EDGS 三角化，比较位置/深度变化，按距离、折叠角、遮挡状态分层。它测敏感性，不证明准确性。`(J^T Sigma^-1 J)^-1` 仅是已知位姿、局部线性及给定观测噪声模型下的一阶协方差近似；不能把 RoMa confidence 直接塞成逆方差，也不能忽略共享参考导致的相关性。
4. **真实几何准确性。** 若有独立注册的深度/点云真值可进一步检查；本次没有核实 Mip-NeRF360 的独立稠密GT，不能将同一COLMAP产生的稀疏点当独立真值，更不能用PSNR替代几何误差。
5. **覆盖与代价。** 同时报 nominal点数、sample footprint、finite/positive-depth覆盖、旧opacity-pass率、验证可观测覆盖、世界/图像空间占用及fallback原因。列出全采样→完整三观测→有效候选→旧opacity资格→成功重排→留出观测可用各层的数量，同时给出每层条件分母和全部参考像素的统一分母。已知可用留出观测上的负深度/无效投影计失败，不能从新方法分母中删掉。只有名义点数和sample footprint是保留原采样即可保证，其余必须测量。另记录纯评分/总初始化耗时与峰值显存。
6. **统计单位。** 像素/track/同次训练checkpoint均相关；按参考相机、scene、独立seed分组才有相应的重复解释。本轮不做显著性检验，也不启动补实验。

## I. 创新性与实施风险

| 不可当作新贡献的已有技术 | 可能形成但尚未证明的 EDGS 专用贡献 |
|---|---|
| 成对重投影、confidence、跨邻居采样聚合：EDGS已有 | 冻结原三候选、固定采样预算下，候选特定的非生成视角支持诊断与选择协议 |
| 多视图一致性用于Gaussian初始化：TVG TUGI已有先例 | 明确区分score视角与未参与选择的训练验证视角，并给出无泄漏证据 |
| 角残差、正深度、三角化角、RANSAC支持数和残差决胜：SfM/VGGSfM已有 | 兼容EDGS旧pair-error→opacity合约的保守回退与有效覆盖审计 |
| 全组3D散度或简单medoid共识：经典一致性指标 | 解释何时相对confidence仍误选，并以独立残差/稳定性/覆盖/代价实证支撑改进 |

仅把现成角残差或TVG散度搬入 `select_best_keypoints()`，创新边界很弱；若没有独立几何证据，最多是启发式重排。反之，“最小改动、coverage-preserving”也不能仅靠工程叙述成立。以上只指出可研究空间，不宣称完成新颖性检索或已有论文证明新方案有效。

其他实际风险包括：RoMa错误观测相关；遮挡状态不明；异尺寸/像素中心约定；把未匹配相机传入VGGSfM的全相机cheirality检查；`nan_to_num`掩盖无效值；支持数不同分母；隐式改变opacity；在验证视角上反复调阈值；全候选失效时覆盖与有效性不可兼得。各项都需显式记录，而不是新增经验权重去掩盖。

## J. 最终建议及审核事项

**推荐保留这个研究方向，暂不批准式推进实现或训练。** 当前最可信的切入点不是新建融合乘积，而是：在同一参考像素的原三候选上，建立正确坐标与可观测性明确的 out-of-pair 几何支持，并用真正未参与选择的训练视角检验其是否有价值。

仍缺的关键证据：

1. 实际候选bundle与坐标闭环数值验证，尤其投影分母约定和异尺寸边界。
2. 独立于选择的训练观测及明确的缺测/遮挡处理；当前旧产物不足。
3. 无新自由融合权重的阈值校准协议、支持判据和回退消融设计。
4. 有效覆盖不下降、几何稳定性/独立观测改善和开销可接受的实证。

若科研总控同意下一步，需另行授权“最小数据导出/初始化级诊断”，并明确是否允许只读数据上运行数值检查；不应直接进入正式训练。**本轮已停止在报告与只读证据层面，未实施模块、未启动新实验，等待科研总控审核。**
