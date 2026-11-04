# EDGS C-Guard：覆盖约束下的多视图几何可靠性受控验证

**状态：机械阶段停止报告；独立验证尚未执行。日期：2026-11-04（北京时间）。**

本轮已经按最终完整指令05实现隔离模块，并在固定30000个采样位置、90000个Corrected候选上完成初步机械检查与独立复算。独立核验的185项严格检查中184项通过，33项数值核验全部通过，A/B/C-Guard索引复算差异为0；但`train_test_split_identity`失败，整体审计结果保留为`false`。依据指令05的机械审计停止要求，未进入最终冻结或新增匹配。

只读诊断确认，唯一失败来自**本轮核验器的整字典相等判断混合了静态相机身份与后补运行信息**，没有发现相机身份、顺序、共有姿态字段或测试列表变化。此处解释失败来源，不把失败记录改成通过，不据此声称C-Guard方案失败或有效。

## A. 环境、输入与哈希

- 服务器：`3090-server`；容器：`edgs-v1`，执行前已运行，检查时无训练/其他实验进程；RTX 3090 24 GiB，约153 MiB占用、利用率0%；工作盘约305 GB可用。
- 项目：`/workspace/projects/EDGS-main`；Counter：`/workspace/datasets/mipnerf360/counter`。
- 参考训练索引20、94、161；每参考10000个原sample IDs，每位置三个原候选。原邻居分别为[19,32,31]、[95,96,93]、[162,160,163]。
- 固定使用指令02缓存`b_xyz`及`max(b_err1,b_err2)`；没有读取Original Geometry结果作为本轮方法输入，没有重新三角化、重新抽样或重跑原邻居匹配。
- 原输入/控制6个NPZ、历史3个selection NPZ经原manifest哈希核验；样本、相机映射、参考二维坐标和候选索引逐项核验。实际210训练/30测试身份分离；没有加载正式测试图像。
- 本轮机械阶段没有打开旧第四视角的holdout/evaluation文件、oracle标签或指令04失败分析数据。1106位置的机制由原三个邻居观测直接重建。

正式源文件前后SHA256全部不变：

| 文件（`source/`） | SHA256 |
| --- | --- |
| corr_init.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |
| corr_init_baseline_seed228.py | 9f3199443473f127c349b926fc3e7d6ff209f17a8cb8854fd8a5cb8be8dc3c10 |
| corr_init_softgeo_l1_tau2.py | 4257d209a4fdbad850c39d69e76f5ad8d44887fc1dc9b81675c7c7a2924ba17b |
| corr_init_unified_v1_frozen.py | 63a90f0b5bcd784d288e58bbc06746b9e2df78f2fb056c5fc4815f1de6d1b410 |
| corr_init_unified_v2_relative_conf.py | 5e04db1c57c21a4b8f81ded657a255e80d051a984461d3461a1bd9263e13ab88 |

本轮代码及协议（尚未生成最终冻结manifest）：

| 文件 | SHA256 |
| --- | --- |
| guard_core.py | d83d441295fcbdf342186b23ed073582242a2cb610cb7f80e86e3f1f4d4689b8 |
| guarded_pilot.py | 547b03d9152d60d844768a1165244844200b00f20f1e699ca60e37c7a88b2287 |
| independent_mechanical_audit.py | f2eb051d84f896d39eacd9a16ef106b6f1290023312bd31ff03a27794b80aa81 |
| guarded_analysis.py | a323ddc544ae8adacf2c751542499c21de5c44df38118003aaff3a38488e2868 |
| instruction05_final_original.txt | 4569337c1eb7440d8613aabd78cd5770c4ea22651e53a5eccc22c54273843970 |

原有14组正式实验的2114个历史文件，其路径、大小、mtime及日志哈希前后相同；未修改正式源码、冻结快照、模型、日志或checkpoint。

## B. 实现的规则与数学口径

独立实现位于新目录的`guard_core.py::choose_guard()`；输入只接受原三候选、原两幅生成观测、三邻居匹配/confidence及相机矩阵，不接受留出观测。输出A/B/C-Guard索引、实际XYZ、原pair E、候选资格、角残差及回退标志。

1. A保持缓存Corrected Baseline原pair E的argmin及首索引tie语义，不额外过滤。
2. B沿用03的生成几何有效性：有限XYZ/E；参考/生成相机正深度；无EPS可计算投影；原`1e-12`数值退化护栏和折叠视差角>0。不增加经验性视差门槛。B无有效候选时回退A。
3. `common_obs_valid`同时检查三个邻居的有限二维坐标、有限且>0的confidence及观测NDC两轴在[-1,1]，与候选索引无关。
4. Guard资格为`common & B_geometry & ref_FOV & generating_FOV & E<=0.01 & out_of_pair_geometry`。非生成评分视角只要求正深度、数值有效投影/射线/角残差，不添加预测FOV硬门槛。
5. 角公式保持03：行向量世界坐标经`XW_j`得到相机坐标；观测bearing为`normalize([u_ndc/Q00,v_ndc/Q11,1])`，预测bearing为`normalize((XW_j).xyz)`；`alpha=acos(clamp(dot(b_obs,b_pred),-1,1))`，评分为两个非生成视角角残差的算术均值（弧度）。几何与评分用float64；原XYZ和E以原float32精确保留。
6. 词典序：平均角残差、精确tie时原E、再精确tie时原NN索引。缺失/非法残差不填0。common无效或没有合格候选直接返回B；B自身无有效候选才沿用B回退A。每位置始终返回原三候选之一。
7. 原opacity读取所选候选的原E，`E>0.01`为不合格，不读取新score。NaN/Inf另记录，不利用比较陷阱宣告合格。

资格不等同经过标定的真实可靠性；两个可评分观测也不等同两个角度inlier。没有confidence融合权重、角度inlier阈值或新的评分公式。

## C. B回归与机械审计

初步GPU机械检查中，三参考A/B索引与03冻结结果完全相同，B的90000个生成几何标志也完全相同。B相对A仅改变11点，修正所选负深度；这是生成有效性检查的效果，不是多视图排序收益。

以下为**已生成、独立逐点复算索引一致的机械结果**，不是完整研究协议通过或独立验证结果：

| 全30000位置 | A Corrected Baseline | B Validity Only | C-Guard |
| --- | --- | --- | --- |
| 名义输出数 | 30000 | 30000 | 30000 |
| 原opacity合格数 | 29951 | 29951 | 29951 |
| 参考/生成相机均正深度 | 29989 | 30000 | 30000 |
| 生成几何有效数 | 29989 | 30000 | 30000 |
| 参考+生成邻居投影FOV均合格 | 29915 | 29915 | 29916 |
| 生成几何/FOV/opacity联合合格 | 29871 | 29882 | 29883 |
| 相对A索引变化 | 0 | 11 | 21815 |
| 相对B索引变化 | 11 | 0 | 21815 |

逐参考结果：

| ref | C相对B改选 | A/B/C opacity合格 | B→C联合合格 | common无效 | common有效但无合格候选 | 进入Guard选择 | 至少2个合格候选 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 20 | 7468 | 10000/10000/10000 | 9986→9986 | 546 | 1 | 9453 | 8536 |
| 94 | 7246 | 9951/9951/9951 | 9933→9933 | 328 | 31 | 9641 | 9414 |
| 161 | 7101 | 10000/10000/10000 | 9963→9964 | 387 | 1 | 9612 | 9596 |

common有效28739/30000，无效1261。C-Guard进入受约束选择28706个位置，其中仅1个合格候选1160个、至少2个候选可比较27546个。最终资格数0/1/2/3的直方图为[1294, 1160, 2969, 24577]。不能把所有“进入Guard”位置称为发生多候选排序收益。

共同观测无效回退1261，common有效但无合格候选回退33，总回退1294；B无有效生成候选的回退为0。原E、XYZ、sample IDs、选中E归属、逐点无新增opacity不合格、共同资格对称及回退优先级等本轮保护检查均通过。

**独立核验整体仍未通过**：`independent_verification.json`保留`all_checks_passed=false`、185严格检查中184通过、33数值检查全通过、精确索引差异0；唯一失败项如下节。

## D. 唯一失败项：元数据schema混合比较

核验器使用了：

```python
split['train'] == oldm['all_training_cameras']
```

作为相机划分身份条件。这要求两个JSON每个字典的字段集合也完全相同，而原记录并非同一schema。

只读诊断结果：

- 两表均210条，顺序相同；每条静态记录的全部14个字段在manifest相应记录中存在且值相同，包含train_idx、runtime UID、COLMAP ID、图像名/路径、R/T、FoV、原图尺寸等。
- 30张测试图像名称列表逐项完全相同。
- 仅索引0、1、19、20、31、32、93、94、95、96、160、161、162、163这14条manifest记录多出`raw_image_sha256`、`runtime_image_sha256`、`runtime_width`、`runtime_height`，是既有加载过程补充的信息；没有共有字段值不同，没有split独有字段。
- 因此整字典比较False是核验器设计问题。目前没有发现相机身份或姿态改变的证据，不能把它描述成缓存损坏或数据泄漏。

此说明不替代重新审计。按05“机械审计失败即停止、先提交问题报告”，**本轮未修改核验器、未重跑审计、未改判通过，也没有开发C-Guard v2**。`metadata_failure_diagnosis.json`保留逐行字段差异，原失败JSON和日志完整保留。

## E. 对原资格不对称与opacity预算的回答

仅用原三个邻居重建原C资格不对称强制改选位置：ref20为476、ref94为301、ref161为329，共1106。C-Guard在这1106个位置全部因共同观测无效返回B，进入角排序0、相对B改选0。

这支持“本数据上原来的候选自身跳过坏生成观测的排序绕过已被消除”；不表示无效观测处被回退输出的B就获得真实几何保证。所有1261个common无效位置都不执行Guard排序。

原opacity合格预算A=B=C-Guard=29951；相对B逐位置新增不合格数0，既有49个不合格点原样存在。生成几何/FOV/opacity联合资格B为29882，C-Guard为29883，逐位置没有丢失原B联合合格点。该保护是当前固定候选上的机械属性，尚不是独立几何收益证据。

## F. 新视角规划与未进入冻结的时间链

prepare已按协议使用实际float32展平world_view矩阵及`torch.cdist(p=2)`，排除参考、原3NN和对应旧第四训练视角，按(distance, train_idx)排序。完整210训练身份、距离矩阵和各ref全部剩余距离序列已保存。

以下**仅是预定身份，尚未加载其图像开展验证匹配**：

| ref | 排除索引 | 计划新train_idx | camera.uid/runtime_uid | COLMAP image_id | image_name | 距离 |
| --- | --- | --- | --- | --- | --- | --- |
| 20 | [18, 19, 20, 31, 32] | 33 | 38 | 39 | DSCF5895.JPG | 0.6169062257 |
| 94 | [92, 93, 94, 95, 96] | 97 | 111 | 107 | DSCF5968.JPG | 0.4699366391 |
| 161 | [160, 161, 162, 163, 171] | 170 | 195 | 191 | DSCF6052.JPG | 0.4020900428 |

- 初步选择/机械结果保存完成：`2026-11-04T01:10:55.658404+00:00`（UTC；北京时间09:10:55）。
- 独立核验失败记录：`2026-11-04T01:11:21.306610+00:00`（UTC；北京时间09:11:21）。
- 停止后完整性审计：`2026-11-04T01:13:36.137404+00:00`（UTC；北京时间09:13:36）。
- **没有`selection_manifest.json`最终冻结manifest，没有`holdout_attempts.jsonl`，新增forward为0，holdout/evaluation文件为0。** `protocol_frozen.json`是prepare保存的规则草稿快照；这个文件名不表示通过外部审计或完成最终方法冻结。
- 没有RoMa模型加载、新增匹配、正式训练、重评或模型保存，也没有读取旧留出结果来决定规则。

## G. 独立验证、分层与有效覆盖：缺失结果明确列示

新留出像素L2、bearing角残差的median/mean/P95/P99/max、配对更好/更差/相等数、新观测有效/缺失率、新视角正深度/FOV以及共同有效统计均**未产生（N/A，非0）**。未执行16×16图像网格及固定32³空间占用的最终覆盖分析。不能将名义30000点或联合资格29883解释为真实三维覆盖不变。

预先确定的分箱已保存在prepare manifest：A原pair E四分位、视差角[0,1,2,5,10,90]度、历史正深度候选四分位、原90k confidence四分位，另有参考帧/改选/回退分组。未根据任何新验证结果选箱或调整规则。后处理脚本只写入、做过合成分母/Inf/tie测试；本轮未运行它读取验证数据。

C-Guard相对A及B均改选21815/30000=72.7167%，仍是大规模改选。**当前不能把这些改选称为错误，也不能称为改善**；需要尚未执行的新独立观测才能判断。原C的失败结果不可以替代新Guard的验证结果。

## H. 已测初始化级开销

下面GPU测量覆盖`choose_guard()`的输入转float64/拷贝、全部候选几何与角计算、A/B/Guard选择和内置保护检查，**不是Guard独占的增量消融成本**；RoMa不驻留。已同步CUDA，无额外匹配warmup。

| ref | 模块秒 | 额外峰值allocated MiB | allocated基线 MiB | 导出/机械记录秒 |
| --- | --- | --- | --- | --- |
| 20 | 0.110933 | 14.713867 | 8.548340 | 0.322389 |
| 94 | 0.010992 | 14.713867 | 8.548340 | 0.323084 |
| 161 | 0.010987 | 14.713867 | 8.548340 | 0.317431 |

三参考模块合计0.132912 s，每参考额外峰值allocated 14.713867 MiB（额外reserved 14 MiB）。首调用明显包含首次相关算子启动成本；后两次约0.011 s，不能把单次微基准外推80参考帧速度。

GPU context初始化0.245320 s，相机元数据与距离阶段1.133342 s，逐参考缓存加载2.272060 s，prepare总时间10.220531 s。导出/机械记录合计0.962904 s。新增3对RoMa耗时和模型加载耗时N/A（未执行，不是测得0 s）。

## I. 可复核数据、脚本、哈希与命令

服务器隔离目录：

`/workspace/projects/EDGS-main/research_experiments/multiview_reliability_guarded_20261104/`

其中`run01/`保存原文、规则快照、输入hash引用、逐候选资格/XYZ/E/角残差、A/B/CGuard索引、fallback和失败审计；顶层保存四个脚本、环境审计、metadata失败诊断、停止完整性审计和stdout/stderr日志。原大缓存仍在原目录，未上传GitHub。

本地完整只读备份：

`D:\TTJ\文档\ChatGPT\3DGS\research_experiments\multiview_reliability_guarded_20261104\server_snapshot\`

完整prepare命令（容器内、项目目录下，`CUDA_VISIBLE_DEVICES=0`）：

```bash
set -o pipefail
python3 -B research_experiments/multiview_reliability_guarded_20261104/guarded_pilot.py \
  --stage prepare \
  --out /workspace/projects/EDGS-main/research_experiments/multiview_reliability_guarded_20261104/run01 \
  --instruction /workspace/projects/EDGS-main/research_experiments/multiview_reliability_guarded_20261104/instruction05_final_original.txt \
  2>&1 | tee research_experiments/multiview_reliability_guarded_20261104/prepare_run01.log
```

prepare退出0。独立核验命令：

```bash
set -o pipefail
python3 -B research_experiments/multiview_reliability_guarded_20261104/independent_mechanical_audit.py \
  --run-dir /workspace/projects/EDGS-main/research_experiments/multiview_reliability_guarded_20261104/run01 \
  --old-dir /workspace/projects/EDGS-main/research_experiments/real_matching_geometry_20261030T051950Z \
  --pilot-dir /workspace/projects/EDGS-main/research_experiments/multiview_reliability_pilot_20261103_run01 \
  --out-json /workspace/projects/EDGS-main/research_experiments/multiview_reliability_guarded_20261104/run01/independent_verification.json \
  2>&1 | tee research_experiments/multiview_reliability_guarded_20261104/independent_audit_run01.log
```

独立核验退出2，pipefail保留失败。实际执行前也检查日志不存在；所有产物使用新目录/排他写入。freeze、validate及post-validation analysis均没有执行，禁止直接重跑已有目录覆盖证据。

关键文件SHA256：

| `run01/`文件 | SHA256 |
| --- | --- |
| prepare_manifest.json | e60f2b6b3a53ea7bc4767144a88849a8dd508ed3465fd105326a39543e878cd3 |
| mechanical_audit.json | c4b63ea60041bf6416d0db6a17d5fb92028bc007accef4d13980a7427025ee6f |
| independent_verification.json | a4db1f40af1c2bbd84f0ecebb929ee229a5a61e3cc14dcfe9c4f59c9cbc29727 |
| selected_ref_020.npz | 0f98de87a5c6464210b62324fbbb2bcc3c782cef6794c93e123eebf16b883407 |
| selected_ref_094.npz | 574e95c94814c9647955bed977183dff6f416f86da101f2ccef16799e9634da7 |
| selected_ref_161.npz | 88e9b8ececf76dcdbf6b269caaedd74c5bb0cd279705ffe1ee8e6d660077e9b8 |
| training_pose_distances.npz | 0f8b934289f3e44c81a57e68d3114aa15760b61f66778e86744e60231048b91f |
| training_camera_identity.json | cec27bf8bd5d7f809f9672ed65d63abef74fe21859f73131438c9227491864d2 |

## J. 科研判断与下一步

| 总控问题 | 当前证据与结论 |
| --- | --- |
| 资格不对称是否解决？ | 固定数据的规则、掩码与索引复算支持消除原排序绕过；1106位置全部返回B。整体审计因核验器schema项未通过，不能标记全部阶段完成。 |
| 是否避免新增opacity预算损失？ | 机械结果为29951，与B相同，逐点新增不合格0；保留原49点。 |
| 相对A/B独立几何一致性是否改善？ | 未知：新增验证0对，没有新的误差统计。 |
| 是否保持有效稠密覆盖？ | 生成几何/FOV/opacity联合资格未降，但尚无新留出及网格/空间代理结果；不能下真实覆盖结论。 |
| 是否仍有大规模错误切换？ | 大规模切换存在（72.7167%），错误性质未知，不能用旧C结论替代新验证。 |

**本轮不能进入指令05的“通过且改善/变差”分支，也不能提出正式接入或训练建议。** 当前停止由核验器的schema检查错误触发，不是证明Guard数学规则无效。传统bearing残差、FOV、正深度与原阈值保护不构成已证实的新几何方法；本轮没有新增可支持论文创新或泛化的独立效果证据。

提交总控审核的最小后续建议：仅修订/版本化核验器的相机身份schema检查，逐行严格比较明确的静态身份、pose、FoV和原尺寸字段；对后补运行尺寸/图像哈希另设独立核验，未知字段不静默忽略。保留本轮失败证据，使用新核验输出，不覆盖旧文件；不改变C-Guard规则、阈值、样本、候选、预定视角或原结果。是否允许重新核验、最终冻结及新增3对匹配，等待总控审核。

停止状态：无正式代码/冻结文件变化，无历史14组结果变化，无新匹配、三角化、训练或重评。执行窗口已停止，不自动进入下一实验。
