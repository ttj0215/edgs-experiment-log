# C-Guard跨场景离线复现：指令07 / 07-W停止报告

**状态：机械审计失败后停止，等待科研总控审核。未完成跨场景独立验证。**

本轮已完成两次获准的历史预热、18对生成匹配、180000个Corrected Geometry候选及60000个位置的A/B/C/D选择。独立机械审计严格检查1920/1938通过、数值检查568/568通过，四法选择索引独立复算差异为0。18项目标坐标归一化字节检查失败，生产流程已停止：没有最终选择冻结manifest，没有六对新留出匹配，没有独立效果、覆盖或尾部评价，也没有训练。

只读排查已数值定位到本轮新增核验器的标量类型错误。服务器NumPy 2.2.6下，核验器使用`np.int64`分母，使float32数组除法产生float64中间结果；冻结EDGS原函数使用图像shape提供的Python `int`。保持原算术路径复算18/18组均与缓存逐字节一致。此诊断不改写原失败结果，也不构成完整机械审计通过或C-Guard有效性结论。

## A. 协议、场景与预定相机

依据本聊天中的科研总控指令07及07-W。07原始预检/相机记录和先前预热授权阻塞事件均保留；07-W明确补充允许每场景train[0]→train[1]一次预热。每场景3个参考帧×10000个位置×3个候选，方法、阈值0.01、原RoMa权重和预处理保持不变。

| 场景 | 训练/测试相机 | 原图尺寸 | 运行尺寸 | 参考索引 |
| --- | --- | --- | --- | --- |
| garden | 161 / 24 | 5187×3361 | 1600×1036 | 32, 80, 128 |
| bonsai | 255 / 37 | 3118×2078 | 1600×1066 | 50, 127, 203 |

参考规则按实际EDGS排序后的训练列表计算`[(N-1)//5,(N-1)//2,(4*(N-1))//5]`。原3NN使用展平float32 `world_view_transform`、原`torch.cdist`与`topk(3)`，保留返回顺序；留出按相同实际姿态向量距离、排除参考和3NN、精确距离平局按train_idx确定。

| 场景 | ref train_idx | UID | COLMAP ID | 参考图像 | 有序3NN | 预定留出（未匹配） |
| --- | --- | --- | --- | --- | --- | --- |
| garden | 32 | 37 | 38 | DSC07993.JPG | 3,33,31 | 2: DSC07959.JPG (UID 3, COLMAP 4) |
| garden | 80 | 92 | 93 | DSC08048.JPG | 53,106,54 | 79: DSC08047.JPG (UID 91, COLMAP 92) |
| garden | 128 | 147 | 148 | DSC08103.JPG | 127,153,129 | 152: DSC08130.JPG (UID 174, COLMAP 175) |
| bonsai | 50 | 58 | 59 | DSCF5623.JPG | 51,49,52 | 48: DSCF5620.JPG (UID 55, COLMAP 56) |
| bonsai | 127 | 146 | 147 | DSCF5711.JPG | 128,126,129 | 130: DSCF5714.JPG (UID 149, COLMAP 150) |
| bonsai | 203 | 233 | 234 | DSCF5798.JPG | 204,202,209 | 201: DSCF5795.JPG (UID 230, COLMAP 231) |

完整训练及测试身份、R/T、FoV、图像路径、原始尺寸、运行尺寸、图像与姿态哈希位于`camera_manifest.json`及各场景`poses.npz`。训练和测试列表分别核验，未读取测试观测进入实验。预定留出训练图像在预检中仅用于身份、尺寸和图像哈希核验，没有产生RoMa留出观测。

## B. 原候选生成与07-W预热

实际沿用历史task02辅助模块（SHA见末尾），通过AST提取冻结的原匹配、resize、confidence-max、采样和坐标转换函数。场景包装仅替换预定参考、实际相机数量及说明文字；新增只读计数/资源挂钩经反向移除后，与原采样流程AST一致。

每场景在原模型初始化前调用原`set_seed(228)`，不按参考或邻居额外重置。每参考帧仅执行一次原`torch.multinomial`，三个邻居共享同一组10000个sample IDs，保存抽样前后RNG、原始/resize warp、confidence、采样匹配、像素转换前值及最终u1/u2。未重抽样。Corrected求解只使用历史已归档的clipW AST：`P1_2/P2_2`列2→3，其余FP32 CUDA最小二乘、原EPS=1e-4 NDC-L1误差均保留；未运行Original Geometry。原`E=max(err1,err2)`单独保存，没有被角分数或物理像素误差替换。

| 调用类别 | 07-W上限 | 实际尝试 | 实际完成 | 重试 |
| --- | --- | --- | --- | --- |
| 历史预热 | 2 | 2 | 2 | 0 |
| 生成邻居匹配 | 18 | 18 | 18 | 0 |
| 新留出匹配 | 6 | 0 | 0 | 0 |
| 合计 | 26 | 20 | 20 | 0 |

每次原forward进入前写入并落盘账本，完成/异常分别记录；同场景第二次预热会在原forward进入前被拒绝。采集账本（含2次预热及18次正式生成）共20个begin及20个complete，身份和顺序核验通过。留出账本不存在。

| 场景 | 预热训练图像 | forward | 同步耗时s | 峰值allocated MiB | 相对进入前新增MiB | 峰值reserved MiB | 实际RNG前后 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| garden | 0 DSC07957.JPG → 1 DSC07958.JPG | 1 | 0.614129464 | 1460.443359 | 972.354004 | 2542 | 未改变 |
| bonsai | 0 DSCF5566.JPG → 1 DSCF5567.JPG | 1 | 0.398468545 | 1464.880859 | 972.866211 | 2550 | 未改变 |

上述时间含原`model.match`及审计挂钩开销，为本次单次测量，不是纯网络基准。两次合计同步预热耗时约1.012598009s。预热前后Python、NumPy、Torch CPU和CUDA RNG状态哈希本次均未变化；读取/保存记录自身不消耗RNG的检查通过。状态数组保存在各场景`warmup_rng07w.npz`。这不推断所有环境的预热均不消耗随机数。

预热的warp/confidence结果沿用原流程只记录形状/耗时随后删除，不进入正式缓存、采样、资格、分数或验证。预热成本与模型加载/候选生成分开，不能算作C-Guard独有开销。

## C. A/B/C/D及机械审计

- A：Corrected候选的原最小pair E，沿用原精确平局及NaN argmin行为。
- B：冻结的Geometry Validity Only及原Baseline回退。
- C：直接导入既有`guard_core.py`，三邻居共同观测资格、生成几何与FOV、E≤0.01、非生成双视角连续平均角残差及原tie/fallback均未修改。
- D：预注册诊断索引，在同一C资格集合内按原E最小选择，无资格或common无效时回退B；不作为新正式算法。

| 审计项目 | 结果 |
| --- | --- |
| 严格检查 | 1920 / 1938通过；18项失败 |
| 数值检查 | 568 / 568通过 |
| A/B/C/D独立索引差异 | 0 |
| 两个场景、六参考、60000位置/180000候选 | 已保存并核验归属 |
| C相对B逐位置新增opacity失败 | 0（该检查通过） |
| 原C资格不对称防护、fallback及原E归属 | 对应检查通过 |
| 最终审计总判定 | 失败，退出码2，保持原判定 |

以下仅为保存选择的机械计数，来自未通过完整审计的本轮产物；不能替代留出结果、有效空间覆盖或科研收益。

| 场景 | ref | C≠B | D≠B | opacity A/B/C/D | 生成几何∩FOV∩opacity A/B/C/D | 三共同观测 | 执行Guard排序（含单候选） |
| --- | --- | --- | --- | --- | --- | --- | --- |
| garden | 32 | 7567 | 0 | 10000/10000/10000/10000 | 9958/9958/9958/9958 | 9722 | 9722 |
| garden | 80 | 7764 | 1 | 10000/10000/10000/10000 | 9959/9959/9960/9960 | 9536 | 9533 |
| garden | 128 | 7326 | 2 | 10000/10000/10000/10000 | 9963/9963/9965/9965 | 9624 | 9624 |
| bonsai | 50 | 7587 | 0 | 10000/10000/10000/10000 | 9990/9990/9990/9990 | 9744 | 9743 |
| bonsai | 127 | 8217 | 2 | 9998/9998/9998/9998 | 9979/9979/9981/9981 | 9790 | 9784 |
| bonsai | 203 | 7200 | 1 | 9988/9988/9988/9988 | 9927/9927/9928/9928 | 9697 | 9690 |

Garden的C≠B为22657/30000（75.5233%），Bonsai为23004/30000（76.6800%）；两场景各D≠B为3点。Garden四法opacity均30000，Bonsai四法均29986。生成联合资格B/C分别为Garden 29880/29883、Bonsai 29896/29899。这些预算检查不证明空间覆盖保持，也不证明大规模改选正确。

## D. 机械失败、只读诊断与时间链

本次中断来自新增独立核验器的实现错误：它在目标坐标归一化中直接使用整数数组元素作分母，没有严格复现原函数的Python标量类型。该错误属于实验基础设施，不能包装为EDGS算法缺陷，也不能把诊断结论当成原审计通过。

原EDGS代码的`H_B,W_B`来自图像`.shape`（Python `int`）：

```python
kptsB_np[:, 0] = kptsB_np[:, 0] / W_B * 2.0 - 1.0
kptsB_np[:, 1] = kptsB_np[:, 1] / H_B * 2.0 - 1.0
```

本次核验器的分母来自`d["dimensions"][j+1,k]`（`np.int64`）。服务器NumPy 2.2.6实测原float32数组除Python `int`产生float32中间结果，除`np.int64`则产生float64中间结果，再赋回float32。两者最终dtype相同，计算路径和逐字节结果仍可不同。仅检查最终dtype或数组形状不足以保证相同计算合约。

| 只读复算路径 | 字节一致组数 | 不同分量总数 | 最大绝对差（NDC） |
| --- | --- | --- | --- |
| 原核验器np.int64分母 | 0 / 18 | 187105 | 1.1920928955078125e-7 |
| 原EDGS Python int分母 | 18 / 18 | 0 | 0 |

18项失败对应两个场景各三个参考、每参考三个目标邻居。所有source归一化检查通过，target全部失败，符合source检查使用shape的Python `int`而target使用数组标量的差异。`failure_diagnosis07w.json`保存全部18组的dtype、不同分量数、极值和代表sample IDs；没有修改输入坐标、候选、索引、核验器或失败JSON。

| 场景 | ref | NN | 除法中间dtype：核验/原 | 核验不同row | 核验不同分量 | 最大NDC差 | 原路径字节一致 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| garden | 32 | 0 | float64 / float32 | 7777 | 10472 | 1.19209289551e-07 | True |
| garden | 32 | 1 | float64 / float32 | 7439 | 9978 | 1.19209289551e-07 | True |
| garden | 32 | 2 | float64 / float32 | 7825 | 10546 | 1.19209289551e-07 | True |
| garden | 80 | 0 | float64 / float32 | 7674 | 10236 | 1.19209289551e-07 | True |
| garden | 80 | 1 | float64 / float32 | 7454 | 9912 | 1.19209289551e-07 | True |
| garden | 80 | 2 | float64 / float32 | 7665 | 10408 | 1.19209289551e-07 | True |
| garden | 128 | 0 | float64 / float32 | 7770 | 10412 | 1.19209289551e-07 | True |
| garden | 128 | 1 | float64 / float32 | 7538 | 10232 | 5.96046447754e-08 | True |
| garden | 128 | 2 | float64 / float32 | 7434 | 10061 | 1.19209289551e-07 | True |
| bonsai | 50 | 0 | float64 / float32 | 7737 | 10477 | 1.19209289551e-07 | True |
| bonsai | 50 | 1 | float64 / float32 | 7398 | 9719 | 1.19209289551e-07 | True |
| bonsai | 50 | 2 | float64 / float32 | 8037 | 11185 | 1.19209289551e-07 | True |
| bonsai | 127 | 0 | float64 / float32 | 7817 | 10533 | 1.19209289551e-07 | True |
| bonsai | 127 | 1 | float64 / float32 | 7554 | 10165 | 1.19209289551e-07 | True |
| bonsai | 127 | 2 | float64 / float32 | 7953 | 10941 | 1.19209289551e-07 | True |
| bonsai | 203 | 0 | float64 / float32 | 7543 | 10175 | 1.19209289551e-07 | True |
| bonsai | 203 | 1 | float64 / float32 | 7841 | 10742 | 1.19209289551e-07 | True |
| bonsai | 203 | 2 | float64 / float32 | 7911 | 10911 | 1.19209289551e-07 | True |

代表样本：garden ref32 NN0、offset 0、sample ID 801651；转换前像素`[3.928232192993164, 576.630859375]`，原缓存NDC`[-0.9950897097587585, 0.11318695545196533]`，核验器复算`[-0.9950897097587585, 0.11318698525428772]`。原Python int路径与缓存完全一致。这个差异是合约复现中的舍入差，不是独立几何精度测量。

**原核验器及原失败结果均未覆盖，未制作强制通过或放宽容差版本，未重跑完整审计。** 若总控后续授权，建议仅令核验器分母明确恢复原Python `int`语义，保留逐字节检查；在实际NumPy 2.2.6环境加入此回归用例，另存新版本及新审计输出，复用已保存的18对匹配和固定选择。此建议尚未实施，仍须完整重审后才能冻结并开始六对留出。

| 阶段 | 开始UTC | 结束UTC | 退出码 | 墙钟s | 完整stdout/stderr |
| --- | --- | --- | --- | --- | --- |
| acquire | 2026-11-04T08:02:56.746423+00:00 | 2026-11-04T08:05:10.383517+00:00 | 0 | 133.637094 | acquire07w.log |
| prepare | 2026-11-04T08:05:54.743750+00:00 | 2026-11-04T08:06:39.811498+00:00 | 0 | 45.067748 | prepare07w.log |
| audit | 2026-11-04T08:07:23.005194+00:00 | 2026-11-04T08:08:24.827203+00:00 | 2 | 61.822009 | audit07w.log |

冻结、留出匹配和独立评价阶段均未启动。`prepare_manifest.json`是待独立审计的准备记录，不等同最终`selection_manifest.json`；后者不存在。预定相机身份冻结与全方法最终选择冻结是两个阶段，不混称为已完成最终冻结。

实际环境：容器`edgs-v1`，RTX3090，执行前GPU153MiB/0%，服务器剩余空间约305GiB；相关记录保持现场原值。每关键阶段完整命令、环境、开始/结束、退出码及日志SHA位于`command_receipts07w`、`stage_receipts07w`。匹配命令在`set -o pipefail`环境下执行，阶段记录器合并完整stdout/stderr并传播子进程真实退出码。

```bash
# cwd: /workspace/projects/EDGS-main
# CUDA_VISIBLE_DEVICES=0
# PYTHONDONTWRITEBYTECODE=1
# PYTHONPATH=/workspace/projects/EDGS-main/submodules/RoMa
python3 research_experiments/guarded_cross_scene_20261104_run01/run_stage_logged07w.py --root /workspace/projects/EDGS-main/research_experiments/guarded_cross_scene_20261104_run01 --stage acquire
python3 research_experiments/guarded_cross_scene_20261104_run01/run_stage_logged07w.py --root /workspace/projects/EDGS-main/research_experiments/guarded_cross_scene_20261104_run01 --stage prepare
python3 research_experiments/guarded_cross_scene_20261104_run01/run_stage_logged07w.py --root /workspace/projects/EDGS-main/research_experiments/guarded_cross_scene_20261104_run01 --stage audit
# audit exit2: STOP. The following phases were NOT executed.
```

## E. Garden独立结果

未产生。三个预定留出视角均未匹配，无留出像素误差、bearing误差、胜负、P99或严重退化结论。不能用生成视图误差、改选率或预算通过代替独立验证。

## F. Bonsai独立结果

未产生。状态与Garden相同。两场景运行尺寸不同，后续若获准恢复，必须按原协议同时报告原像素、像素/运行图像对角线、0.5×NDC-L2以及bearing角，不能拼接不等价像素尺度。

## G. 两场景综合比较

本轮只证明在既定相机和原流程下，两个固定候选池与预注册选择已保存，且相应机械检查结果如上。跨参考帧收益一致性、某场景整体退化、是否可接受的覆盖代价，目前均未知。不得输出正面、负面或混合的科研效果判决。

## H. 资格与角排序归因

D索引已预先生成并保存，但未使用任何新留出误差分析B→D→C收益。D≠B各3点仅是机械差异计数，不是资格阶段贡献；也不能据此把其余改选全部称为净收益。未做oracle、阈值调整或新公式开发。

## I. 空间覆盖与长尾

每场景完整候选池的有限XYZ min/max AABB及预定分层已写入prepare记录，尚未完成最终审计冻结。16³/32³/64³/128³覆盖、双向最近邻、局部密度、深度/视差层及严重留出尾部分析均未运行。不能把60000行、预算计数或AABB存在解释为有效表面覆盖保持。空间单位未标定，不能换算为米。

## J. 数据、哈希、局限与审核建议

服务器隔离目录：`/workspace/projects/EDGS-main/research_experiments/guarded_cross_scene_20261104_run01/`。原始01—06和历史14组正式实验不写入。停止后重新检查414份受保护研究/源码文件的SHA，以及2114个正式输出文件的大小/mtime和日志SHA，均保持不变；准备记录绑定的产物也未变化。原审计的输入前后哈希检查通过。

文件组织：

- 各场景`input_ref_*.npz`：原匹配/resize匹配、置信度、sample IDs、原坐标和相机矩阵、抽样RNG。
- `corrected_ref_*.npz`：三个原Corrected候选XYZ、err1/err2、A索引及原E。
- `selected_ref_*.npz`：冻结核心输出的几何/观测资格、角分数、回退及A/B/C/D索引、原XYZ/E归属。这里是已保存准备结果，未称作最终冻结。
- `warmup_record07w.json`、`warmup_rng07w.npz`、生成forward账本与预算：两次预热和20次实测调用。
- `audit07_v1/independent_audit.json`及`.log`：原1938+568检查，失败保留。
- `failure_diagnosis07w.json`、`stop_status07w.json`、`report_summary07w.json`：只读原因证据和停止状态。
- `data_manifest07w.json`：停止快照所有既存文件的SHA/字节数，不含自身递归哈希。
- `analyze_cross_scene_07.py`仅完成合成自检，未对本轮真实缓存运行。没有evaluation NPZ或analysis07结果目录。

| 关键对象 | SHA256 |
| --- | --- |
| C-Guard核心 | d83d441295fcbdf342186b23ed073582242a2cb610cb7f80e86e3f1f4d4689b8 |
| 历史task02 helper | 080f4b2f2865d15f9be74024be1ddaaeaeeb47c6025932f2ec1899d002891bfc |
| 实际07-W计数包装器 | 742d88d498eb3b520bc4a40bf89572bb7ac2086943f761cce010e2abe21457cd |
| 独立核验器（失败版本保留） | 74c10fd760b45dd9abd6240133c1dadbabab793f42136754b1f105e3fe305e2b |
| 原失败审计JSON | b9f269b793ded376c9eb3e8139795ff69da16855b89f94151077133994020ac2 |
| 只读失败诊断 | 27f0158e5de907c2e8a4f1c3069f06f3e1154b90497df46053c5ba5ce57bcf7a |
| 相机manifest | 6a63dadf6fe05e11b59a7271cdc46f26995270d26d3048732358a32ce6182657 |
| 准备manifest | 6e873729632960eef09cd7a2eae598ebc8c34c9f578d32ff0c4167328d4c4e47 |
| 07协议 | fe0f8692b180db98e9ccdacf51ae238487fec9255121213c271507f96ac97c1e |
| 07-W授权记录 | 664b2e9bfba3baef962b8529ef989d6b56ea316fe5cc3e1dc844dbdb8907f541 |

| 正式源文件 | SHA256 |
| --- | --- |
| corr_init.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |
| corr_init_baseline_seed228.py | 9f3199443473f127c349b926fc3e7d6ff209f17a8cb8854fd8a5cb8be8dc3c10 |
| corr_init_softgeo_l1_tau2.py | 4257d209a4fdbad850c39d69e76f5ad8d44887fc1dc9b81675c7c7a2924ba17b |
| corr_init_unified_v1_frozen.py | 63a90f0b5bcd784d288e58bbc06746b9e2df78f2fb056c5fc4815f1de6d1b410 |
| corr_init_unified_v2_relative_conf.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |

| 场景 | ref | 类型 | 文件 | SHA256 |
| --- | --- | --- | --- | --- |
| garden | 32 | input | garden/input_ref_032.npz | 8348d88ad9325c74213a08cdca0250db283e1c6fabca3401c6f37bccf6428595 |
| garden | 32 | candidate | garden/corrected_ref_032.npz | 08c6be7d1787558b6e71350f250cd0eb07f7cddd7d658320ab049d2f1e656359 |
| garden | 32 | selected | garden/selected_ref_032.npz | 22c8f6c0919003a64892343d797255ce84adf18ae1c1e64550387e6e156f0122 |
| garden | 80 | input | garden/input_ref_080.npz | d8b635bc05f34b0db76cb1177b7eb6179518bc960dc940fd590de6d0b75bc640 |
| garden | 80 | candidate | garden/corrected_ref_080.npz | 9da2eb2e3efdeb34632a5c5a1725d0d823fc4e3f884cfa0c252210e6b554c7ac |
| garden | 80 | selected | garden/selected_ref_080.npz | ff9c7753c8aca3cb349803b71097a6fc7b5a77da07f3e39c794b9f89045435f9 |
| garden | 128 | input | garden/input_ref_128.npz | dbd90010f1775d60d2df03bce9628bd7f22bdc929c05134b5adbdb78c48cde5b |
| garden | 128 | candidate | garden/corrected_ref_128.npz | f866352801eba72392444d01cf3de8722f2a6c26bc9f84349ae69071a1cdc3f1 |
| garden | 128 | selected | garden/selected_ref_128.npz | 8d05d21d6ce06f61b1de2ce73e20342088cf98fd7f2ba5c3507a760e395899c2 |
| bonsai | 50 | input | bonsai/input_ref_050.npz | 32a346e0f421bc0c3755c07886361a9cb10b8ef8c7db7cdae30bd9d7b1b1d7c9 |
| bonsai | 50 | candidate | bonsai/corrected_ref_050.npz | 2a27df8b567b9e00bd7a1ef54ff4478389cbc498b59bc98d4c973aa40cd89ff7 |
| bonsai | 50 | selected | bonsai/selected_ref_050.npz | 6f531b513403bc1c828c4f007efede3713867b3693b717f4b87e00eda750fed1 |
| bonsai | 127 | input | bonsai/input_ref_127.npz | baf0e4a0a24ed9a426921c6733c4c91d80b4968e46417c97aca5373cac1375b6 |
| bonsai | 127 | candidate | bonsai/corrected_ref_127.npz | 48b27fb2630e260314b27f68aa645dbfc0b0bdf94e2aee8d24d807e548581c43 |
| bonsai | 127 | selected | bonsai/selected_ref_127.npz | 4ad26cd6f40dce2baf5e8cbc9550f6f4926f4c310a907e6fd0f84c93ea008315 |
| bonsai | 203 | input | bonsai/input_ref_203.npz | ad10bd55f0eccccbaafe2bc1f2d6bcc54f11ac51113cd626cfca47749f94c535 |
| bonsai | 203 | candidate | bonsai/corrected_ref_203.npz | aa4a640f627a604f8f9d6d121b8e650b95b8715c8e1e340e1d78d6cb52cf2423 |
| bonsai | 203 | selected | bonsai/selected_ref_203.npz | d3529c5e2e346e69e7db8cb171887a46f85d27a52f257c75436739f6c1a4ad0f |

停止快照归档：服务器`/workspace/projects/EDGS-main/research_experiments/guarded_cross_scene_20261104_stopped07w.tar`，576634880字节，SHA256 `858004649d5dc6191e5dd0488c9cc6b0f2f04573c8acfb40d44c8edcc1495f27`。仅上传本Markdown至GitHub，大缓存不上传。

**本地备份状态：停止时为‘本地备份尚未完成’，原停止记录保持该历史状态。随后同步恢复，已在2026-11-04T08:20:27.079117+00:00核对完整快照89个文件及归档SHA。** 本地目录`D:\TTJ\文档\ChatGPT\3DGS\research_experiments\guarded_cross_scene_20261104_run01\server_stop_snapshot07w`；核验回执`D:\TTJ\文档\ChatGPT\3DGS\research_experiments\guarded_cross_scene_20261104_run01\local_backup_verification07w.json`。后续报告/交付回执单独列清单，未覆盖停止快照。

已确认的原因是本次核验器的中间dtype不一致；未确认的是新留出上的任何C-Guard收益、候选误选或覆盖变化。现有失败不证明C-Guard跨场景失败，同样不能根据其他机械项通过宣称C-Guard有效。

本轮没有三维真值或训练/渲染评价；即便后续离线验证完成，也不能直接推断真实三维精度、可见表面完整性或PSNR改善。固定seed及每场景三个相关参考帧不构成多次独立随机实验，30000个相关采样位置不能用作统计显著性的独立重复。

**建议总控仅审核核验器标量类型修复与条件恢复方案；暂不进入正式EDGS接入、训练或其他场景。执行端已停止，不自动修复、重审、重匹配或进入下一阶段。**
