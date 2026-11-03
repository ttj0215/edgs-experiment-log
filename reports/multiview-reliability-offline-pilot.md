# EDGS 多视图几何可靠性：最小侵入式离线试验

日期：2026-11-03（Asia/Shanghai）。科研总控指令03。**已完成一次预先冻结的三参考帧试验；当前 C 原型不建议进入正式 EDGS 接入或完整训练。**

在完全相同的 Corrected Geometry 候选和 30000 个参考采样位置上，非生成双视图连续角残差原型 C 更换候选 24222 次（80.74%），但三个参考帧的留出视角像素误差中位数和均值均恶化。三方法共同有效 N=29679 时，像素中位数 A/B=0.929788、C=1.359535 px；P95=125.4464→127.5522 px；角残差中位数=0.0305967→0.0450519°。原 opacity 门槛不通过数由49增至2174。没有据验证结果改权重、阈值、资格条件或聚合器。

本报告保留局部正面结果：B/C 均消除 A 的11个负深度选择；ref94、ref161 的 C 像素 P95 下降，合并最大误差也下降。它们不足以抵消整体中位数/均值与 opacity 资格的负结果。未使用正式测试图像，没有新增三角化、正式训练、Gaussian 保存或 Unified v3。

## A. 文献与本实现的对应关系

| 依据 | 已核实机制与版本 | 本试验采用/未采用 |
|---|---|---|
| VGGSfM，CVPR2024 | 官方 main e1d9d2eb2b3575525792206fb94b2c749c58dc50（2025-03-11），角残差是单位观测 bearing 与候选相机方向夹角；另有 folded 三角化角、正深度、阈值内点支持数与残差决胜 | 采用已知角几何的独立 PyTorch 数学实现；不移植系统，不使用其三角化/LO/BA，不照搬默认2°角内点阈值、1.5°最小视差或confidence阈值 |
| VGGSfM 版本边界 | 原论文 learned triangulator 与后续 main 的 RANSAC/LO 实现不同；main 用全部track观测评分，含生成视图 | 本原型仅平均两个非生成视图，并要求完整可评分观测；这是我方预注册候选选择设计，不能称官方原算法或论文已证明有效 |
| TVG-SLAM，arXiv2506.23207v1 | tri-view 一致性与初始化不确定性用于完整Gaussian SLAM；官方 main 6f84b0b94b198b93e39e6497920814c662408374（2026-02-23）仍只有README和assets | 仅借鉴研究思想，未复制模块；无公开核心实现/运行配置/LICENSE，不能称可直接移植或有EDGS专用验证 |

可复核来源：[VGGSfM 论文](https://arxiv.org/abs/2312.04563)、[角残差源码 L431–472](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation_helpers.py#L431-L472)、[三角化角 L475–523](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/utils/triangulation_helpers.py#L475-L523)、[残差 indicator L63–87](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/vggsfm/two_view_geo/utils.py#L63-L87)、[TVG-SLAM 论文](https://arxiv.org/abs/2506.23207)、[官方 TVG-SLAM tree](https://github.com/MagicTZ/TVG-SLAM/tree/6f84b0b94b198b93e39e6497920814c662408374)。

VGGSfM [LICENSE.txt](https://github.com/facebookresearch/vggsfm/blob/e1d9d2eb2b3575525792206fb94b2c749c58dc50/LICENSE.txt) 为 CC BY-NC 4.0（非 MIT）。本试验未复制或导入其函数源码，按经典公式独立实现并记录出处；若未来直接复用源码须遵守该许可。完整官方核查见归档 literature-audit.md。TVG-SLAM README 声称 RA-L 接收；IEEE最终页面本轮受验证限制，论文公式/实验依据明确采用 arXiv v1，没有假装核对最终期刊全文。

## B. 环境、缓存完整性与冻结顺序

现场 `edgs-v1` 原处于停止状态，确认默认启动命令为 NVIDIA entrypoint+bash 后恢复容器，没有触发训练。RTX3090、PyTorch2.7.1+cu118、driver580.178.04，容器和GPU空闲，可用磁盘约306GiB。正式项目 `/workspace/projects/EDGS-main`，Counter数据、210训练/30测试划分、原相机和预处理沿用上轮；测试图像未打开，测试相机仅有元数据名单用于排除检查。

固定输入为 `research_experiments/real_matching_geometry_20261030T051950Z/` 的三个 input_ref 和 control_ref 文件。六份缓存 SHA256、五份正式/冻结代码、旧相机/RoMa预处理依赖与两份权重哈希均复核一致。只使用 control 中的 `b_xyz/b_err1/b_err2/b_idx`；Original Geometry 的数组未参与评分、选点或评价。既有三个邻居的候选没有重算，sample IDs 没有重抽样。

过程先后：prepare 仅使用旧缓存，保存协议、每点 A/B/C 索引/坐标/原pair E、相机列表和哈希；随后 validate 验证这些哈希，在不调用选择器的路径上新增3对留出匹配。冻结 UTC=2026-11-03T12:28:43.879349+00:00（北京时间20:28:43）；全部匹配与评价结束 UTC=2026-11-03T12:30:41.835632+00:00（20:30:41）。没有先看留出结果再制定规则。

| 对象 | SHA256 |
|---|---|
| 原型 offline_reliability_pilot.py | 9ed8d0524731c739c579f03130fe9b643a589503f45471031a4824b21f39a489 |
| 运行规范化 protocol_frozen.json | 285654213ca03f40dc45490c581408ce250070b2ffc6283c935ccb14e5b88645 |
| 旧 run manifest.json | 04e2e90d729c250e27a4dd26603b3d9e6fe86d82a7b0e0531b275de8295e94a5 |
| 冻结选择 selection_manifest.json | 90bb793a03849daf7b697efb5814879b52aca4e5a65524d48e3518b6900c6a24 |
| 离线分析 analyze_pilot.py | 85087a88bdfdad7cee8df7ffaa4a83e955a8abdfb638c4af1ffa36a163322a03 |

旧缓存明细哈希位于 manifest.old_input_hashes；新匹配/评价文件和选择哈希位于 manifest.matching/selected。独立验证确认候选90k与每方法选中XYZ、原pair E、sample IDs、相机索引一致。339项严格检查、36项数值检查通过；NumPy独立重现全部 A/B/C 索引和回退，0个选择差异。留出像素误差/相机z复算最大差0；角残差最大差3.99e-10rad。最终以 verification_r2.json 为准；首轮验证器曾因NumPy int32/Torch int64计数类型不同报6项严格差异，已在验证器修正，首轮记录保留，实验数组未改。

## C. 数学式、有效性与独立模块位置

独立脚本在 `/workspace/projects/EDGS-main/research_experiments/offline_reliability_pilot.py`；关键函数 `project_and_angle()`、`folded_angle()`、`choose_fixed()`、`prepare()`、`validate()`。没有修改 `source/corr_init.py`。评分入口只接旧候选/匹配字典，不接留出视角参数；validate只消费已冻结的选择。

EDGS行向量约定：`c_v=X_h W_v`，`clip_v=X_h P_v`，`P_v=W_v Q_v`。已断言实际Q为居中pinhole投影、clipW=z_cam；NDC观测转换到相机bearing为 `b_v=normalize([u_ndc/Q00,v_ndc/Q11,1])`。不能把RoMa/renderer NDC直接作为VGGSfM已去内参的二维输入。

角残差采用已有公式：$\alpha_{iv}=\arccos(\mathrm{clamp}(\hat b_v^T\widehat{c_v(X_i)},-1,1))$。normalize 的 norm 下限 EPS=1e-12，来自官方函数所用 F.normalize 默认值；原型另把零/非有限相机射线明确标成NaN/无效。计算用FP64，输出弧度，报告转度；固定候选XYZ和原pair E数值不变。

退化诊断采用已有 law-of-cosines folded 角：$D=2\sqrt{\|X-C_1\|^2\|X-C_2\|^2}$，$\theta=\arccos(\mathrm{clamp}((d_1^2+d_2^2-b^2)/D,-1,1))$，$\beta=\min(\theta,\pi-\theta)180/\pi$。D≤1e-12或非有限标无效，β须>0。这个长度乘积护栏具有场景单位依赖；官方在小分母时返回0，本原型显式返回NaN+invalid。未使用1.5°等物理质量门，不能声称已筛掉所有近退化小视差点。

状态分开保存：①观测存在=二维有限、confidence有限且>0；②可用=存在且观测NDC在[-1,1]；③候选有效=XYZ/原E有限、生成reference+NN正深度和可计算投影、数值非退化；④已有可用观测下，候选在非生成view负深度/不可投影是候选矛盾，不能记成匹配缺失；⑤有限但大的角残差是连续不一致程度，不是missing。

本原型不设角内点阈值，不进行训练数据阈值校准；confidence只做数值存在性检查，不作权重或可信真值。count=2表示两个可评分额外观测，**不是两个几何内点**。任何有限大角残差都可能进入排序，故不存在经过校准的绝对“高角误差不可靠”拒绝规则；无资格候选时的回退不能被描述为已解决全部错误匹配/遮挡。

## D. 三种候选选择的公平对照

| 方法 | 冻结决策 | 回退/原误差 |
|---|---|---|
| A Corrected Baseline | 直接复用原 Corrected argmin max(err_ref,err_gen)，包括原NumPy tie行为 | 原pair E为EPS=1e-4的NDC-L1；不重新定义selector |
| B Geometry Validity Only | 仅在C节生成几何有效候选中最小化同一个原E | 全部无效则回退A，标invalid fallback；不使用非生成残差 |
| C Multi-View Reliability | 先与B相同生成几何有效；两个非生成NN观测均可用，且候选在两view正深度、可投影、角有限，才可评分；最小化两个角的算术平均 | 精确角平局按原E、再NN index；无eligible则直接回退A并记录原因。保留被选候选原E供opacity，不用角score替代 |

C相对B同时加入完整额外观测资格和连续残差排序；本实验不能把C−B单独归因于“均值排序”这一因素。不同候选排除的生成NN不同，虽然均有2个评分view，视图难度仍可能不同。本协议是一次固定组合原型，不是VGGSfM阈值支持数机制的复现。

| 方法 | 名义点数 | 相对A切换 | 切换率 | 原opacity通过 | 通过率 | 回退 |
|---|---|---|---|---|---|---|
| A Corrected Baseline | 30000 | 0 | 0.0000% | 29951 | 99.8367% | 0 |
| B Validity Only | 30000 | 11 | 0.0367% | 29951 | 99.8367% | 0 |
| C Multi-View Reliability | 30000 | 24222 | 80.7400% | 27826 | 92.7533% | 100 |

opacity只统计旧合约 `bad=(selected_E>0.01)`，原logit资格0或−10；没有创建Gaussian、改变颜色/scale/opacity，也没有删点。全部原E有限，NaN误通过数为0。C新增2125个E>0.01点，原49个仍不通过，无bad→pass；因此名义30000行并没有保持同等的opacity门槛合格预算。

## E. 独立训练视角验证

第4邻居仅从原210训练相机选取：使用原展平world-view矩阵的全pair torch.cdist，排除ref和其冻结3邻居，取最近剩余相机，精确距离平局取train_idx较小者。不是相机中心距离；规则和结果均在匹配前锁定。

| 参考 train_idx / 图像 | 原3个生成/评分NN | 仅验证 train_idx / 图像 | 固定pose距离 | 可用留出观测 /10000 |
|---|---|---|---|---|
| 20 | 19,32,31 | 18 / DSCF5878.JPG | 0.556519 | 9976 |
| 94 | 95,96,93 | 92 / DSCF5963.JPG | 0.4215833 | 9892 |
| 161 | 162,160,163 | 171 / DSCF6053.JPG | 0.3908349 | 9822 |

参考图像依次为 DSCF5880、DSCF5965、DSCF6042；第4图像为 DSCF5878、DSCF5963、DSCF6053，均不在正式测试集。实际 model.forward wrapper 计数=3，预热匹配=0。model.eval()+no_grad恢复原warm-up后推理状态，沿用相同RoMa权重、560×560变换、原Camera双resize/量化路径和1600×1066旧采样网格。

每对新匹配仅执行一次。原raw560源网格、resize后源网格、sampled源坐标、原extract归一化前pixel和最终reference NDC均按dtype/shape/字节哈希与旧数据一致；没有仅凭shape推断对齐。新视角不参与原confidence-max采样、评分、阈值、回退或选点。

独立评价是无EPS物理像素L2和同一bearing角残差。valid=留出观测可用+被选候选生成几何有效+在留出相机正深度/可投影+两残差有限；**不按候选是否投影在FOV内筛主指标**，出界预测必须保留大误差。310个留出观测NDC出界，另A有11个负深度：这些位置仍保留在全部30000分母及失败统计。

### 三方法共同有效子集：N=29679

| 方法 | 指标 | 方法可评估 / 该队列 N | 中位数 | P95 | P99 | 最大值 | 均值 |
|---|---|---|---|---|---|---|---|
| A Corrected Baseline | 像素 L2 / px | 29679 / 29679 | 0.9297878 | 125.4464 | 224.5717 | 352.8806 | 17.79091 |
| A Corrected Baseline | 角残差 / ° | 29679 / 29679 | 0.03059669 | 3.767758 | 6.564027 | 9.934244 | 0.5347673 |
| B Validity Only | 像素 L2 / px | 29679 / 29679 | 0.9297878 | 125.4464 | 224.5717 | 352.8806 | 17.79091 |
| B Validity Only | 角残差 / ° | 29679 / 29679 | 0.03059669 | 3.767758 | 6.564027 | 9.934244 | 0.5347673 |
| C Multi-View Reliability | 像素 L2 / px | 29679 / 29679 | 1.359535 | 127.5522 | 224.6102 | 337.1986 | 20.40634 |
| C Multi-View Reliability | 角残差 / ° | 29679 / 29679 | 0.04505189 | 3.897196 | 6.638059 | 8.77406 | 0.6186903 |

此队列上A/B逐点完全相同。C−A像素均值差+2.615437px，配对差中位数0、P95=+19.08090、P99=+96.83509、最大+262.26394px；中位数差为0与边际中位数上升不矛盾，因有5702个平局。配对像素更低10505、更高13472、相等5702；角更低10516、更高13461、相等5702。所有分母是同一个29679，不把像素当独立重复实验。

### 全部固定位置：N=30000，无法评价记作+Inf的已注册损失口径

| 方法 | 指标 | 方法可评估 / 该队列 N | 中位数 | P95 | P99 | 最大值 | 均值 |
|---|---|---|---|---|---|---|---|
| A Corrected Baseline | 像素 L2 / px | 29679 / 30000 | 0.9582713 | 144.3923 | +Inf | +Inf | +Inf |
| A Corrected Baseline | 角残差 / ° | 29679 / 30000 | 0.03152924 | 4.319515 | +Inf | +Inf | +Inf |
| B Validity Only | 像素 L2 / px | 29690 / 30000 | 0.9582713 | 143.5583 | +Inf | +Inf | +Inf |
| B Validity Only | 角残差 / ° | 29690 / 30000 | 0.03152924 | 4.312974 | +Inf | +Inf | +Inf |
| C Multi-View Reliability | 像素 L2 / px | 29690 / 30000 | 1.409676 | 145.8129 | +Inf | +Inf | +Inf |
| C Multi-View Reliability | 角残差 / ° | 29690 / 30000 | 0.04679698 | 4.415637 | +Inf | +Inf | +Inf |

A的321个+Inf=310个观测出界+11个负深度；B/C的310个+Inf均来自观测出界。这里+Inf是为保留全分母而设定的评价惩罚，不是原始计算溢出或三角化失败。三者P99/最大/均值均+Inf，不能拿有限子集均值冒充全体均值。双方法都无法评价的310个位置标为unordered，不计0差或“胜出平局”。

### 候选无关的可用留出观测队列：N=29690

| 方法 | 指标 | 方法可评估 / 该队列 N | 中位数 | P95 | P99 | 最大值 | 均值 |
|---|---|---|---|---|---|---|---|
| A Corrected Baseline | 像素 L2 / px | 29679 / 29690 | 0.9307672 | 125.6534 | 227.4739 | +Inf | +Inf |
| A Corrected Baseline | 角残差 / ° | 29679 / 29690 | 0.03062719 | 3.786894 | 6.621367 | +Inf | +Inf |
| B Validity Only | 像素 L2 / px | 29690 / 29690 | 0.9307672 | 125.5139 | 224.5699 | 352.8806 | 17.83221 |
| B Validity Only | 角残差 / ° | 29690 / 29690 | 0.03062719 | 3.780153 | 6.563959 | 9.934244 | 0.5361339 |
| C Multi-View Reliability | 像素 L2 / px | 29690 / 29690 | 1.362154 | 127.5379 | 224.5911 | 337.1986 | 20.43289 |
| C Multi-View Reliability | 角残差 / ° | 29690 / 29690 | 0.04508487 | 3.89682 | 6.637573 | 8.77406 | 0.6195236 |

此队列B/C全部可评价，A仍有11个无效点。直接C−B的最大共同有效N=29690：像素更低10516、更高13472、相等5702，平均差+2.600678px；角更低10527、更高13461、相等5702，平均差+0.08338963°。三方法共同有效29679与B/C最大共同有效29690不能混用。summary另保留每方法自身有效子集统计与对应分母，不据不同子集制造改善。

### 逐参考帧：同一个三方法共同有效子集

| ref | 方法 | 共同 N | 中位数 px | P95 px | P99 px | 最大 px | 均值 px |
|---|---|---|---|---|---|---|---|
| 20 | A Corrected Baseline | 9976 | 0.8512178 | 30.45361 | 85.0122 | 190.055 | 6.037072 |
| 20 | C Multi-View Reliability | 9976 | 1.833837 | 66.06287 | 163.257 | 265.1826 | 12.80375 |
| 94 | A Corrected Baseline | 9892 | 0.7469022 | 128.535 | 191.5213 | 221.6899 | 18.57948 |
| 94 | C Multi-View Reliability | 9892 | 0.9439645 | 117.7829 | 190.9134 | 222.7311 | 19.34007 |
| 161 | A Corrected Baseline | 9811 | 1.341371 | 185.7095 | 279.4928 | 352.8806 | 28.94733 |
| 161 | C Multi-View Reliability | 9811 | 1.539919 | 183.1398 | 275.1096 | 337.1986 | 29.21187 |

ref94与ref161的P95下降、合并最大值352.881→337.199px是实际局部改善；但三个ref的中位数和均值全部变差，ref20 P95从30.454升至66.063px。不能选择性忽略这些负结果，也不能仅凭最大值下降宣称可靠性提高。

## F. 失败、回退、有效覆盖与共同分层

| 候选/观测审计 | 数量 / 原始分母 |
|---|---|
| 生成几何有效候选 | 89813 / 90000 |
| 两个额外观测均可评分的候选 | 87200 / 90000 |
| 每参考像素eligible候选数 0 / 1 / 2 / 3 | {'0': 100, '1': 1161, '2': 178, '3': 28561} |
| 非生成评分位点观测不存在 | 0 / 180000 |
| 观测存在但观测NDC出界 | 2722 / 180000 |
| 已有可用额外观测但候选投影/正深度失败 | 368 / 180000 |
| 可用且投影OK后的角非有限 | 0 / 180000 |
| 留出数值观测存在 | 30000 / 30000 |
| 留出可用（存在且NDC在图内） | 29690 / 30000（98.9667%） |
| 留出不存在 | 0 / 30000 |
| 留出存在但出界不可用 | 310 / 30000（1.0333%） |

RoMa dense输出数值存在不等于对应正确、无遮挡或是几何真值；所有留出confidence都有限且>0，未把低confidence自动当可靠。310个是“已预测但图像范围外”，不是无匹配或已证实遮挡；真实遮挡/匹配错误状态未知。缺失/不可用值没有当作0角。

| 资格 / 异常 | A /30000 | B /30000 | C /30000 |
|---|---|---|---|
| 所选生成几何有效 | 29989 | 30000 | 30000 |
| reference非正深度 | 11 | 0 | 0 |
| 生成NN非正深度 | 11 | 0 | 0 |
| 留出非正深度 | 11 | 0 | 0 |
| 留出投影计算失败 | 0 | 0 | 0 |
| XYZ非有限 | 0 | 0 | 0 |
| reference投影出界 | 7 | 7 | 28 |
| 生成NN投影出界 | 79 | 79 | 2277 |
| 留出投影出界（留出观测可用内） | 3476 | 3476 | 3512 |
| 原opacity不通过 | 49 | 49 | 2174 |

B/C修复了11个负深度选择，但C的生成NN投影出界由79增至2277，reference出界由7增至28；留出预测出界3476→3512。投影“可计算”不等于图像内或可见。C共100/30000=0.3333%回退，均为额外观测不完整，ref20/94/161分别60/17/23；无allgeninvalid回退、无额外投影失败回退、无角非有限回退。B回退0。回退均使用A原索引/原E，不利用留出观测决定回退。

| 有效覆盖/预算代理 | A | B | C |
|---|---|---|---|
| 名义参考采样位置 | 30000 | 30000 | 30000 |
| 生成几何+两生成视图FOV+原opacity同时合格 | 29871 | 29882 | 27137 |
| 16×16源图网格占据单元（ref20/94/161，全部输入） | 254/256/256 | 254/256/256 | 254/256/256 |
| 源图网格占据（各方法有效留出点） | 254/256/255 | 254/256/255 | 254/256/255 |
| 共同AABB32³占据体素（全部有限XYZ） | 36 | 30 | 19 |
| 共同AABB32³占据体素（生成几何有效） | 30 | 30 | 19 |

三维网格边界取全部固定Corrected候选的AABB，冻结前制定，范围较大且受极值影响；占据体素减少不能直接解释为真实表面completeness下降或提升。名义像素网格占据不变也不能证明有效空间覆盖保持。每格密度、体素重叠/Jaccard及方法有效子集均在coverage CSV中。基于原合约可检验的生成几何/FOV/opacity联合资格，A/B/C分别为[29871, 29882, 27137]；这是资格交集，不是真实可见点数。

### 共同锚点分层：独立留出像素误差

所有方法共用Corrected Baseline选中点的folded三角化角和reference z作锚点；未按各自结果重分箱。深度边界只用旧Corrected正深度候选四分位，非留出校准；均为COLMAP场景坐标单位，不能称米。以下表采用三方共同有效子集，B与A逐点相同；完整CSV还保留每箱全部原分母、+Inf惩罚、每方法有效数与角残差。

按固定Baseline三角化角（°）：

| 共同分箱 | 共同N | A中位数 | C中位数 | A P95 | C P95 | A均值 | C均值 |
|---|---|---|---|---|---|---|---|
| [0.0,1.0) | 600 | 74.39131 | 67.25596 | 301.5658 | 299.9383 | 104.3916 | 103.0931 |
| [1.0,2.0) | 3359 | 2.305214 | 5.336409 | 211.469 | 211.4805 | 37.62289 | 42.28574 |
| [2.0,5.0) | 15021 | 0.9476407 | 1.213808 | 110.4447 | 105.9541 | 15.68964 | 16.34831 |
| [5.0,10.0) | 8496 | 0.5479833 | 1.029905 | 86.3623 | 118.2606 | 10.99272 | 17.16277 |
| [10.0,90.0] | 2203 | 1.288123 | 1.300254 | 22.00686 | 23.21336 | 4.511068 | 4.704185 |

按固定Baseline参考深度（场景单位）：

| 共同分箱 | 共同N | A中位数 | C中位数 | A P95 | C P95 | A均值 | C均值 |
|---|---|---|---|---|---|---|---|
| [0.009404345,3.249397) | 7922 | 0.7260553 | 1.781721 | 125.4635 | 139.2424 | 16.88814 | 24.19272 |
| [3.249397,3.706913) | 7351 | 0.5555678 | 0.5786913 | 19.0401 | 28.96365 | 6.020205 | 7.160007 |
| [3.706913,4.862602) | 7124 | 1.135683 | 1.375532 | 42.62236 | 37.52257 | 7.554183 | 7.12551 |
| [4.862602,524.0466] | 7282 | 2.353755 | 4.277515 | 209.4628 | 209.4083 | 40.66986 | 42.65172 |

最低角度箱存在局部改善，其余多个箱恶化；这是一组固定相关样本的描述，不据此选择新角阈值或调整方法。各箱之外/非有限anchor也在CSV单列，未静默丢弃。

### 可复核代表样本

下表保留未施加valid掩码或+Inf惩罚的原始诊断值。负深度点即使有有限像素误差仍不可评价，例如A_negative_depth行的A=178.4214px，在全体惩罚表中计为+Inf，不能当作有效匹配误差。

| ref / offset / sample ID | 原因 | A/B/C NN索引 | A像素误差 | C像素误差 | A原E | C原E | C回退码 |
|---|---|---|---|---|---|---|---|
| 20 / 6830 / 1699193 | largest_increase | [0, 0, 2] | 2.918614 | 265.1826 | 0.0009202957 | 0.3361466 | 0 |
| 20 / 4921 / 29320 | largest_decrease | [0, 0, 1] | 116.9903 | 11.81851 | 0.0006867051 | 0.003805816 | 0 |
| 20 / 86 / 1601498 | fallback | [0, 0, 0] | 0.2939488 | 0.2939488 | 8.392334e-05 | 8.392334e-05 | 2 |
| 94 / 4291 / 7995 | largest_increase | [2, 2, 0] | 6.069534 | 140.3765 | 0.0005841851 | 0.07577384 | 0 |
| 94 / 7046 / 127153 | largest_decrease | [0, 0, 2] | 128.8918 | 1.449429 | 0.0002248306 | 0.0003835782 | 0 |
| 94 / 55 / 1646348 | fallback | [1, 1, 1] | 94.9995 | 94.9995 | 0.01291186 | 0.01291186 | 2 |
| 161 / 4140 / 748479 | largest_increase | [1, 1, 0] | 0.4612136 | 158.0883 | 0.0001695305 | 0.0002866685 | 0 |
| 161 / 8132 / 170541 | largest_decrease | [2, 2, 1] | 164.0352 | 20.29398 | 0.0003407896 | 0.003820375 | 0 |
| 161 / 512 / 1083214 | fallback | [1, 1, 1] | 2.253815 | 2.253815 | 2.989173e-05 | 2.989173e-05 | 2 |
| 161 / 1420 / 151273 | A_negative_depth | [2, 0, 1] | 178.4214 | 70.75523 | 0.0004644766 | 0.002198696 | 0 |

这些是按预定诊断原因（最大增加/下降、回退、A负深度）列出的实例，不代替总体统计；每一个原样本的索引、角/像素残差、原E、fallback和mask均保存在point_results.csv与NPZ中。

## G. 计算时间、复杂度与显存

| ref | 可靠性/三方法决策模块 s | 额外峰值allocated MiB | 总峰值allocated MiB | 新RoMa配对 s | 留出评价 s |
|---|---|---|---|---|---|
| 20 | 0.09475169 | 13.93896 | 22.4873 | 0.7652514 | 0.004437079 |
| 94 | 0.01063882 | 13.93896 | 22.4873 | 0.2215403 | 0.001580697 |
| 161 | 0.01035266 | 13.93896 | 22.4873 | 0.2209161 | 0.001444217 |

模块三ref总计0.115743s；首批0.094752s包含首次CUDA算子成本，后两批约0.0106/0.0104s。峰值额外allocated为14,616,064 bytes≈13.94MiB/参考帧，总allocated峰值约22.49MiB；reserved峰值和基线另存manifest。测量时RoMa未常驻，使用torch.cuda同步和重置峰值；模块口径含CPU数组转GPU、所有候选角/有效性及三方法决策，不含输入解压、最终导出或完整EDGS初始化。

新增3对RoMa总计1.207708s，单独模型加载9.303376s，加载阶段总allocated峰值约2027.845MiB。留出评价三批总计0.007461993s。匹配仅作验证，其权重/图片显存不可混称可靠性模块常驻开销。以上为单次测量，不做速度显著性或正式80ref全流程开销承诺。

复杂度为O(R·K²·M)（对每候选投到各NN；K=3、M=10000），评分实际用每候选2个非生成view；逐ref批量PyTorch计算，不产生新X，当前rich诊断张量的显存为O(K²M)。没有移植VGGSfM RANSAC/LO或TVG-SLAM系统。

## H. 支持与不支持的证据

| 结论范围 | 本次证据 |
|---|---|
| 支持基础几何有效性检查的必要性 | A原最小pair-error会选11个负深度点；B仅更换11点并恢复正深度，原opacity门槛无法自行排除它们 |
| 不支持当前C的独立一致性收益 | 三ref中位数/均值均差；pooled median/P95/mean与角残差恶化；直接C−B同样为负，不能用生成/评分view的较小残差作成功证据 |
| 有限的正面局部结果 | ref94/161的P95、pooled最大误差和低角度箱部分指标改善；不据此忽略总体负结果 |
| 不支持有效覆盖/opacity预算保持 | 名义30000保持，但opacity合格29951→27826，生成NN出界79→2277；粗网格保持不等于真实覆盖保持 |
| 未证实因果解释 | 逐对RoMa观测未必构成正确多视图track；遮挡/低confidence/评分view难度差异、缺少生成误差约束可能相关，但当前实验没有隔离证明任何一个是主要原因 |

留出仅指该视角未参与本次评分/调参，不是统计独立的随机实验。它共享参考图、相机估计、RoMa模型和邻近场景，匹配可能错误；残差只是跨视角一致性代理，不是真实三维精度。30000个密集像素、3个参考帧与各分层均有关联，未计算p值或宣称统计显著性。没有用此负结果否定所有可靠性感知方向，只否定当前预注册连续组合原型的充分正面证据。

## I. 可能贡献与创新重合风险

传统bearing角误差、正深度、三角化退化、支持观测数、残差排序与多视图一致性均为已有技术；VGGSfM已有相关机制，TVG-SLAM已有tri-view与初始化不确定性思想，不能将它们重新命名后声称原创。

可能值得继续研究的EDGS专用问题是：固定RoMa候选预算下、生成/评分/验证视角分离、显式缺测和资格回退、原opacity误差合约保留，以及实际有效覆盖与效率的联合审查。这是潜在问题定义和适配贡献，不是本轮已证明有效的新算法；当前C组合出现负结果，尚不能支撑论文主创新或SPL有效性结论。

## J. 决策、归档与停止

**暂不建议把当前C原型接入正式EDGS或开始完整训练。** 本轮按一次固定协议结束，没有改mean为max/median、增加confidence权重、引入角阈值、根据留出结果筛点或尝试Unified v3。B的小规模几何有效性结果可供总控后续单独审查，但本次同样不实施正式修复。

如总控后续重新授权，可先研究已有三邻居观测是否形成可信track，以及在不牺牲原pair-error/opacity资格的前提下能否定义有训练数据依据的可靠性判据；这只是后续建议，未开发、未跑新方案。

服务器归档位置：

```text
/workspace/projects/EDGS-main/research_experiments/multiview_reliability_pilot_20261103_run01/
/workspace/projects/EDGS-main/research_experiments/offline_reliability_pilot.py
/workspace/projects/EDGS-main/research_experiments/pilot_protocol.json
/workspace/projects/EDGS-main/research_experiments/pilot03_prepare_20261103_run01.log
/workspace/projects/EDGS-main/research_experiments/pilot03_validate_20261103_run01.log
```

run中保留旧数据引用与哈希、新raw/resize匹配、sample IDs、相机ID/矩阵、选择冻结文件、逐候选角/状态和逐方法留出评价；后续analysis_delivery/保存报告、分析/验证脚本、完整CSV/JSON、文献与协议审查、原始日志及交付哈希。宿主对应 `/home/ttj/workspace/projects/EDGS-main/research_experiments/`，本地完整副本为 `D:/TTJ/文档/ChatGPT/3DGS/research_experiments/multiview_reliability_pilot_20261103/`。大体积数据留服务器/本地，不上传GitHub完整缓存。

| 新文件 | SHA256 |
|---|---|
| selected_ref_020.npz | 4acd2d53d967e4c4e5d60757c7b5728a7655a4942da405481e70cd407ece5ce7 |
| selected_ref_094.npz | e51182ec3bb1dd229213c4c9c20df89bbc87665c16578bc6c81935ddf3af16c3 |
| selected_ref_161.npz | 2f5d34e0459cfb60dbcbfdae334853777c7063f6a8d44dfe6d7e391c18648a53 |
| holdout_ref_020.npz | c44f990ec53b2a70ea70aa2026923cffefcddfc359684d9796b93a0e95941daa |
| holdout_ref_094.npz | a0e8c20e2dbc0e1350892c87b9c6e2ad2411efd26cd316ce513b6a82191a4b8e |
| holdout_ref_161.npz | 44e38e26cc604e5264faa1828804497ac5db512c994b2f2df287146cca1529c6 |
| evaluation_ref_020.npz | 99eb282b25a95b9e594cdf6defc2c7776f8d8b1b03ff7034e6d035855fb6c146 |
| evaluation_ref_094.npz | a0289a12e496d4a908e7ad49966c95ac47962bb01bf5b2e7d87aa54e102809af |
| evaluation_ref_161.npz | cabf3bdceb4a10d1b7a4a68ebc1b0821e00ead1bc4b296240e18dfb3f7032609 |

执行命令（宿主bash，日志事先不存在，均使用set -o pipefail并完整tee stdout/stderr；两阶段退出码均0）：

```bash
set -o pipefail
docker exec -e CUDA_VISIBLE_DEVICES=0 -e PYTHONDONTWRITEBYTECODE=1 \
  -w /workspace/projects/EDGS-main edgs-v1 python3 -u -B \
  research_experiments/offline_reliability_pilot.py --stage prepare \
  --protocol research_experiments/pilot_protocol.json \
  --out /workspace/projects/EDGS-main/research_experiments/multiview_reliability_pilot_20261103_run01 \
  2>&1 | tee /home/ttj/workspace/projects/EDGS-main/research_experiments/pilot03_prepare_20261103_run01.log

docker exec -e CUDA_VISIBLE_DEVICES=0 -e PYTHONDONTWRITEBYTECODE=1 \
  -w /workspace/projects/EDGS-main edgs-v1 python3 -u -B \
  research_experiments/offline_reliability_pilot.py --stage validate \
  --out /workspace/projects/EDGS-main/research_experiments/multiview_reliability_pilot_20261103_run01 \
  2>&1 | tee /home/ttj/workspace/projects/EDGS-main/research_experiments/pilot03_validate_20261103_run01.log
```

独立分析 `analyze_pilot.py --run-dir RUN`；独立验证 `verify_pilot.py` 只用NumPy，不调用原型/匹配/三角化。归档同时保留参数用法、分析SHA、每输入哈希、先后时间链和错误记录。基础合成单元检查只测试投影/角/缺测/回退，不额外读取场景图像或运行匹配。

正式文件运行前后SHA均相同：

| source文件 | SHA256 |
|---|---|
| corr_init.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |
| corr_init_baseline_seed228.py | 9f3199443473f127c349b926fc3e7d6ff209f17a8cb8854fd8a5cb8be8dc3c10 |
| corr_init_softgeo_l1_tau2.py | 4257d209a4fdbad850c39d69e76f5ad8d44887fc1dc9b81675c7c7a2924ba17b |
| corr_init_unified_v1_frozen.py | 63a90f0b5bcd784d288e58bbc06746b9e2df78f2fb056c5fc4815f1de6d1b410 |
| corr_init_unified_v2_relative_conf.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |

2114个历史output文件的size/mtime与所有.log SHA前后相同，旧六份缓存保持不变；没有加载trainer/Gaussian/renderer模块，未启动正式30k训练、重评、其他场景或模型保存。Github仅新增本报告，既有14组结果和历史报告不改。完成后停止，等待科研总控审核。
