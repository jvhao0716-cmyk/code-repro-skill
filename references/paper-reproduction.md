# 论文与机器学习实验复现门禁

本参考用于论文、深度学习模型和 benchmark 复现。目标是先证明环境与执行路径可用，再启动正式实验，避免“运行—报错—补包—继续正式跑”的循环。它不限定框架或论文；GPU、指标、数据集、模型权重等仅在目标实际需要时设为必需项。

## 阶段规则

按顺序执行 Stage 0–9。每个适用阶段记录 PASS 或 FAIL；只有目标确实不需要时才可标 N/A，并写明理由。缺少证据不能算 PASS。任一必需阶段失败，都不进入下一阶段的正式运行。

| 阶段 | 通过条件 |
|---|---|
| 0. 论文与实验定义 | 确定官方论文/仓库/源码 commit、目标表格或行为、配置、输入、运行入口、成功证据和指标协议 |
| 1. 资源预检 | 所需数据、checkpoint、evaluator 文件结构正确；磁盘、网络、硬件和驱动满足目标要求 |
| 2. 官方依赖审计 | 汇总官方安装规格，已安装或计划安装的关键包均满足声明的版本范围 |
| 3. 环境与深层导入 | 环境快照已保存；外部命令可用或已选 fallback；逐层 import 和关键模块检查通过 |
| 4. 模型实例化 | 从项目配置构建目标 model/pipeline，或成功完成等价的“加载但不推理”流程 |
| 5. 单样本 smoke test | baseline 单样本通过；目标方法单样本通过且核心机制有正向证据 |
| 6. 正式 baseline | 官方 baseline/no-cache reference 按冻结配置完成，产物和记录完整 |
| 7. Proposed method | 提出的方法正式运行成功，方法参数和核心机制证据已记录 |
| 8. Metrics/evaluator | evaluator 的输入、reference protocol 和指标定义明确；只评估成功运行 |
| 9. Ablation | baseline、proposed method 和 metrics pipeline 均稳定；代码与配置已冻结并可区分 |

Stage 6–9 是正式实验阶段。Stage 0–5 通过前，不要启动正式 baseline、批量推理、指标汇总或 ablation。

## Stage 0：建立复现契约

先从论文、官方仓库和配置文件确认：

- 要复现的表格、图、行为或基线；官方实现的仓库、commit/tag 和入口。
- 数据、checkpoint、evaluator、运行参数、随机种子、硬件要求及官方 reference。
- 唯一成功证据，以及每项命令或阶段的预期输出、PASS/FAIL 条件和失败后是否停止。
- baseline、proposed method、metrics 和 ablation 各自的运行范围。

在正式实验前建立 experiment manifest。至少记录 experiment_id、method、sample/input、prompt（适用时）、seed、resolution、frames、steps、guidance、checkpoint、git commit、environment、algorithm parameters 和 start time。无法确定的值标为 unknown，不用论文表格中的结果替代本次运行记录。

## Stage 1：资源与主机预检

检查目标项目路径中的数据、模型、权重分片、配置、tokenizer、benchmark 和 evaluator 文件是否已存在；按官方说明检查目录结构、必要文件和可校验信息。然后核对所需磁盘空间、网络来源、GPU/驱动/CUDA 及操作系统条件。只有目标依赖 GPU 时才要求 GPU 检查通过。

下载前顺序：

1. 检查目标路径、目录结构和必需文件。
2. 检查目标磁盘空间。
3. 单独测试官方来源网络；网络测试与实际下载分开。
4. 官方来源失败后才测试允许的镜像，并保留失败证据。
5. 使用支持续传的工具下载，校验文件完整性。
6. 解压后重新检查预期文件和结构。

全新实例声明仍按 Skill 主文件的快速模式处理：只检查项目目标路径和执行路径确实需要的主机条件，不扫描无关磁盘、全局缓存或完整系统状态。

## Stage 2：官方依赖审计与安装

安装前先阅读适用的官方 requirements.txt、pyproject.toml、environment.yml、setup.py/setup.cfg、README 安装说明、子模块和模型组件 requirements。建立一份依赖清单，涵盖目标执行路径、关键模块和 evaluator；标出 Python、PyTorch、CUDA、NumPy、OpenCV、Transformers、Diffusers、Accelerate 等关键包（只列本项目实际使用的项）。

逐项确认当前版本满足官方约束范围。例如 numpy>=1.26.4,<2.0.0 要直接比较实际安装版本；pip check 通过不能代替范围核对或官方兼容性确认。把 pip check 当作一个检查项。

缺依赖或遇到 ImportError 时，不要立刻单独安装报错包并重跑正式实验。返回依赖审计，确认该包属于哪个官方依赖，分析安装会否改变 NumPy、Torch、OpenCV 等关键版本；需要时用 constraints 锁住关键版本。安装后重新执行相关依赖审计、环境快照和 Deep Import Check，不能把安装成功视为 gate 通过。

## Stage 3：环境快照、外部工具和 Deep Import Check

正式 baseline 前保存 env_before.txt；依赖或系统环境变更后保存 env_final.txt。记录与目标相关的 Python、pip/conda、关键包版本、Torch 版本、Torch CUDA 版本、GPU 名称、driver、CUDA 是否可用；使用 Conda 时记录 conda list，使用 pip 时记录 pip freeze。避免导出或公开 token、密码和其他敏感环境变量。

按层验证导入，前一层通过后再继续：

1. 基础包：如 torch、目标模型库和评估库。
2. 项目模块：真实导入论文代码中的 utils、pipeline、模型、dataloader 等。
3. adapter 和子组件：检查实际入口会加载的 adapter、custom op、evaluator 和相关模块。
4. 从项目配置构造模型所需的模块/类；模型实例化本身留在 Stage 4。

基础 import torch 或 import transformers 通过，不代表项目深层依赖可用。任何缺失都回到 Stage 2；不要边跑正式实验边补包。

脚本使用 time、ffmpeg、git、wget、curl、nvidia-smi、unzip、aria2c 等外部命令时，先用当前 shell 的等价命令检查其是否存在（Unix 可用 command -v，Windows PowerShell 可用 Get-Command）。缺少辅助命令时先选稳定的等价方案；计时可用应用内的单调时钟，不要默认 /usr/bin/time 一定存在。记录检查命令的预期输出、PASS/FAIL 和是否阻止后续阶段。

## Stage 4：实例化模型或 pipeline

使用正式配置和目标 checkpoint 执行真正的 config load 与 model/pipeline instantiate，或项目提供的等价“load model but no inference”路径。检查必要权重、组件、tokenizer、adapter 和 device placement 均已加载。只完成浅层 import 不算通过；实例化失败时归类为环境/依赖或模型/文件加载问题，不归因于算法效果。

## Stage 5：单样本 smoke test

正式 N 样本运行前先跑 baseline/reference × 1。只有退出码为 0，输出文件存在且结构/帧数符合预期，必要日志和 metadata 存在，延迟与显存数据来自真实推理区间，且目标要求 GPU 时确认推理实际使用 GPU，才可通过。

随后用相同代表性输入跑 proposed method × 1。除上述条件外，还要用日志、计数器、instrumentation 或调用路径证明核心机制真正触发（例如 cache hit、skip 或 prediction path 大于零；以目标算法定义为准）。无触发证据就停止在 smoke test 阶段，不能开始批量方法实验。

## Stage 6–7：Baseline 与 proposed method 正式运行

先运行 official/original/no-cache baseline，再运行完整 proposed method。执行前冻结论文规定的参数、输入、seed 和代码 revision；若为复现诊断而要修改官方代码，先保留原始 commit 和 clean baseline，并把修改放在可追踪的 branch/worktree/patch 中。记录 baseline 与 modified code SHA，避免把实验性改动误认为官方实现。

每个 run 使用独立临时目录，例如 runs/tmp/<experiment-id>/<run-id>，状态从 PENDING 开始。只有 exit_code == 0、期望产物及数量正确、必要 metadata/log 存在时，才能标记 SUCCESS 并进入 aggregate metrics。否则标记 FAILED，记录失败阶段、退出码和错误日志；失败 run 不得写入正式聚合指标。

Latency 只记录成功推理阶段内实际测得的时间。模型初始化失败后耗时 4 秒等总运行时间不能作为 inference latency。显存和 GPU 指标也应在推理期间记录：按项目可用性保存 peak allocated/reserved、nvidia-smi 峰值或 GPU utilization，并说明测量方式；不可得时明确标注，不能事后反推。

## Stage 8：Metrics 与 evaluator

正式生成前定义要计算的指标、输入配对、预处理和 evaluator 版本。对于 PSNR、SSIM、LPIPS、WorldScore 等指标，先从论文或官方代码确认 reference protocol：例如 generated output 对 ground truth、cached output 对 no-cache output，或其他明确配对。将 metric_reference_type 和具体 reference 路径写入 metadata。

reference 不存在或协议无法确认时，将相关指标标记为 unavailable 并记录原因。只把 SUCCESS run 输入 evaluator；严禁复制论文表格数值充作本次复现结果。

## Stage 9：Ablation

只有 baseline、proposed method 和 metrics pipeline 都稳定且结果记录完整后，才开始 ablation。先冻结当前代码与配置，记录 commit/hash，再通过明确的配置开关逐项改变实验因素。每个消融项写入独立 manifest/run，并说明相对完整方法改变了什么。Baseline 未成功或核心算法仍在修补时，先完成 clean baseline，不进入消融。

## 统一 Preflight Gate 与错误分类

正式 baseline 的 ALLOW_FORMAL_EXPERIMENT 只有在以下适用项全部通过时才可设为 TRUE：

- [ ] Stage 0 的源码版本、配置、输入、成功证据和 manifest 已明确。
- [ ] 所需文件、目录结构、磁盘和来源可用。
- [ ] 目标需要的硬件、驱动和 CUDA 可用。
- [ ] 官方依赖和版本范围满足；pip check 等辅助检查已记录。
- [ ] 关键包版本与环境快照已保存。
- [ ] 目标项目深层 import、外部命令或 fallback 均已验证。
- [ ] 模型/pipeline instantiate 通过。
- [ ] 输出目录可写，临时 run 和失败隔离规则已就绪。
- [ ] baseline 单样本通过；将运行的算法方法单样本通过且机制已触发。

对目标明确不需要的检查，记录 N/A 和理由。任一适用项失败或未知时保持 ALLOW_FORMAL_EXPERIMENT = FALSE。

将问题明确归为以下一类并先在该层修复：

- **环境/依赖问题：**版本不满足、系统命令缺失、CUDA/驱动/权限或资源不足。
- **模型/文件加载问题：**checkpoint、配置、权重结构、组件或实例化失败。
- **算法/代码问题：**在模型与输入已有效加载后，目标方法路径错误或机制未触发。
- **实验结果问题：**运行有效且成功，但输出、指标或统计结果不满足论文判据。

某一类问题的失败不能直接宣告另一类失败；每次修复后只重跑受影响的门禁，并按顺序恢复流程。

