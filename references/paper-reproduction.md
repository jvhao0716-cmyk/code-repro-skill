# 论文、模型与 benchmark 复现

用于论文代码、深度学习模型、GPU 实验、benchmark、metrics、evaluator 和 ablation。把一次复现视为范围冻结、只读审计、验证、正式生成和独立评估组成的流程。不得把某个项目的参数或工具名写成所有项目的固定要求。

## 先冻结任务范围和指标

在安装或实验前，确认并记录：

- 用户要复现的结果/行为，以及明确不需要或排除的工作。
- 论文、官方仓库、commit/tag、入口与目标表格/图/样本。
- 指定的模型/backbone、checkpoint、实验组、样本、数据集或 evaluation set。
- resolution、frames、steps、seed、guidance、算法参数。
- 最终指标、reference protocol、evaluator 和最终产物。

不能仅因论文包含多个模型/实验就全部运行。不能擅自添加 baseline、original、额外 backbone、benchmark、指标或 ablation。Baseline 只有在用户范围包含或目标协议需要时才纳入；若科学有效性似乎需要用户未指定的 baseline，先说明并确认是否扩展范围。用户明确排除的项目是硬约束；若排除项使某个比较/指标无法成立，应保留排除并报告该限制，不自行加跑。

把最终表格或交付物需要的每项指标前置到 generation 规划，并反推其输入、reference、帧对齐、resize、normalization、aggregation、evaluator 版本和 runtime instrumentation。若 reference/evaluator 缺失或协议不确定，应在 generation 前报告，不能等生成后再发现指标不可算。将配置集中写入 experiment_manifest.yaml、config.yaml 或等价文件，避免依赖临时 shell export 或人工记忆。

先判定项目是训练、微调、纯 inference 还是 training-free 方法。Training-free 任务通常需要 evaluation set/benchmark/input samples/reference，不要自动要求 training set。

## 阶段门禁

按顺序记录每个适用阶段的 PASS、FAIL 或带理由的 N/A。任何未知项不得视为 PASS。阶段名称可以按项目调整，但不得绕过硬门禁或改变用户冻结的范围。

| 阶段 | 工作与通过条件 |
|---|---|
| 0. Scope/metric freeze | 用户范围、排除项、实验组、成功证据、所有指标及 evaluator/reference protocol 已写入 manifest |
| 1. Read-only Preflight | 系统、Python、GPU/CUDA、依赖规格、extensions、checkpoint、数据、网络、磁盘、输出/日志/resume 条件已审计；首次审计不修改环境 |
| 2. Error Audit | 当前所有错误、阻塞 warning 和缺项已提取，按共同根因与阻塞关系归组；一次性修复计划已形成 |
| 3. Batched minimal repair | 只修复确认的问题；PASS 核心组件保持冻结；互不依赖的问题合并修复，遮蔽项明确分阶段 |
| 4. Final validation/freeze | 官方版本范围、依赖、CUDA/native backend、checkpoint/data、load-only model/pipeline instantiate、输出、日志和恢复能力均通过；保存环境快照，设置 ENV_READY/READY_TO_RUN |
| 5. Smoke test OR N/A | 用户未拒绝时按目标真实路径跑最小代表样本；用户明确拒绝时记录 N/A，并将首个正式样本事务化 |
| 6. Scoped generation | 仅运行冻结范围中的实验组/样本；baseline/reference 仅在 scope 或协议要求时纳入；每个 sample 独立事务、持久记录、可恢复；昂贵 GPU 开始前 READY_FOR_PAID_GPU |
| 7. Bundle validation/resume | 每个 SUCCESS/DONE 的 Evaluation Bundle 满足完整性判据；FAILED 项与 aggregate metrics 隔离，恢复时跳过有效 DONE |
| 8. Evaluation/metrics | 只评估 SUCCESS bundle；按冻结的 reference protocol 在适当的 evaluator 环境执行，并保存可复跑评测记录 |
| 9. Requested ablation/report | 仅运行用户要求或协议确实需要的 ablation；除被研究因素外保持条件一致，报告可追溯证据 |

Stage 1 是只读阶段：禁止安装、升级、改配置、下载大文件或启动昂贵正式实验。此阶段需要审计每个新接入的大模型、world model、evaluator、benchmark、CUDA extension 的真实调用路径；不要只靠经验猜依赖。Stage 2 和 Stage 3 的细则见 [error-audit-and-recovery.md](error-audit-and-recovery.md)。

## Stage 1：目标相关只读预检

检查范围由冻结的执行路径决定，记录目标不需要的项为 N/A：

- **系统资源：**OS/kernel、CPU、RAM/swap、目标文件系统和磁盘空间。
- **GPU/CUDA：**GPU 型号/专用显存、driver、CUDA runtime/toolkit、nvcc、CUDA_HOME、cuDNN（若需要）；检查 torch/torchvision/torchaudio、torch CUDA 和 CUDA availability。目标要求时做真实 CUDA tensor 运算，不只读版本字符串。
- **Python 路径：**Python/Conda/pip 版本、当前 environment、python executable、site-packages、PATH、PYTHONPATH、LD_LIBRARY_PATH 等相关变量；不导出 secrets。
- **官方依赖规格：**读取 requirements、environment.yml、pyproject、setup 配置、README、submodule、模型和 evaluator 的依赖；追踪真实 import/call path。
- **CUDA/native extensions：**package 存在不等于 backend 可用。若路径调用 FlashAttention、xformers、DROID、lietorch、torch_scatter、custom op 等，检查编译产物、动态库加载及真实 backend/function 调用。
- **模型与数据：**按执行路径检查 checkpoint、config、tokenizer、VAE、transformer、text encoder、adapter、scheduler、weight shards、evaluation samples、prompt、image/video、camera 和 benchmark/reference metadata 的存在、结构、权限与合理大小。
- **运行基础设施：**目标输出目录可写；日志位置、cache/persistent disk、tmux/screen/job scheduler 和 resume 状态机制可用。
- **下载条件：**检查目标路径、已有资源/cache、可用磁盘和官方源网络。网络探测与下载分开；有证据表明官方源不可达后再测允许的镜像。

预检摘要应清楚给出适用项状态，并以 READY_TO_RUN=YES/NO 收尾，例如：

    SYSTEM=PASS
    GPU=PASS or N/A (reason)
    CUDA=PASS or N/A (reason)
    PYTHON=PASS
    PYTORCH=PASS
    DEPENDENCIES=PASS/FAIL
    CUDA_EXTENSIONS=PASS/FAIL/N/A
    CHECKPOINTS=PASS/FAIL/N/A
    DATASET=PASS/FAIL/N/A
    DISK=PASS/FAIL
    OUTPUT=PASS/FAIL
    LOGGER=PASS/FAIL
    RESUME=PASS/FAIL
    READY_TO_RUN=YES/NO

所有必需项通过前不得开始正式昂贵 GPU 实验。CPU 任务不因无 GPU 被阻塞；GPU 任务不因版本显示正确而跳过实际 CUDA/backend 检查。

## Stage 2–4：审计、最小修复和冻结

先完整读取现有错误信息，再按共同根因修复；不要按报错行逐条补包。完整修复方案细则、分阶段条件、生成/评测环境隔离、日志和环境冻结见 [error-audit-and-recovery.md](error-audit-and-recovery.md)。

首次只读审计后，生成环境和 evaluation/benchmark 环境若依赖冲突或 evaluator 缺少额外组件，应分开管理（例如 model_gen_env 与 evaluator_env），不得为 evaluator 任意破坏已通过的生成环境。正式验证通过后记录 ENV_READY=YES 并冻结环境；以后只运行。遇到新错误时停止当前 run，收集日志，重新审计并计划针对性修复，再重新验证和更新快照。

保存与复现有关的快照：pip freeze、conda list（适用时）、environment.yml/lock、Python/Torch/CUDA/driver、关键依赖版本、git commit、checkpoint identifier。避免保存密钥或不必要的个人环境信息。环境变更后生成新的 final snapshot。

## Stage 4–5：Load-only 与可选 smoke test

Final validation/freeze 阶段按目标配置真实加载 checkpoint 和 model/pipeline；验证必需权重、components、device placement 和 evaluator 初始化（适用时）。这不是用 random weights 或 synthetic input 代替用户的真实模型实验。只有这些 load-only 检查通过，才能设置 ENV_READY=YES。

Smoke test 是降低风险的工具，不凌驾于用户的实验协议。若用户明确说不做 smoke test，Stage 5 记为 N/A (user explicitly declined)。仍须通过官方依赖/版本审计、load-only model validation、checkpoint/data validation、输出/log/resume validation。随后只把第一个正式样本作为事务式运行；该样本失败就停止，不启动剩余 batch。

若用户未拒绝 smoke test，使用能触达相同核心路径的真实最小输入。仅当不会改变目标行为时才缩小 resolution/frames/steps；不得换成随机权重、fake prompt、synthetic tensor 或另一种实现。Baseline/reference smoke test 只在冻结范围要求 baseline/reference 时运行。Proposed method 要有机制触发证据；如 cache hit/skip/prediction counters 或目标函数调用路径。

## Stage 7：Generation、evaluation bundle 与恢复

正式 generation 前，除 experiment manifest 外，为每个 sample/run 建立独立临时目录。每个 bundle 按目标需要保存：

    sample_xxx/
      output.mp4 or target output
      frames/                 # evaluator needs them
      input/ and input metadata
      prompt/camera metadata  # when applicable
      generation_config
      algorithm_config
      runtime_metrics.json
      environment snapshot/reference
      run.log
      status.json
      DONE or FAILED

generation 只执行一次；后续 evaluator 可从 bundle 重复运行，不应重做 generation。

每个 sample 独立处理：开始前检查有效 DONE；成功后验证并持久化产物，再写 DONE；失败写 FAILED、退出码、失败阶段和日志，不进入 aggregate metrics。恢复时跳过有效 DONE，只重试失败/不完整 sample。DONE 不只依据文件名：还要检查 exit_code=0、输出存在且大小合理、结构/帧数正确、metadata 和 runtime log 完整。

在真实推理区间记录 inference start/end、latency、torch.cuda.max_memory_allocated/reserved、可用时 NVML peak memory、需要时 GPU utilization、GPU model、config 和 environment。失败初始化耗时不得计作 latency；不可测指标写 unavailable 和原因，不事后反推。

正式长任务必须在 tmux/screen、systemd-run、scheduler 或平台可靠 job system 等断连保护机制内运行，stdout/stderr 同时持久写磁盘。SSH/Jupyter/VS Code/browser session 断开后，任务和日志必须继续。安装、编译、下载、inference 和 evaluator 同样保留完整 stdout/stderr；排错优先读取完整日志，而非只看 traceback 最后几行。

在付费 GPU 实例启动前尽量完成读代码、scope/manifest、依赖审计、样本选择、指标协议、输出 schema、脚本、错误与恢复计划。昂贵 GPU 上只执行必须使用它的验证/运行。开机前 gate 使用 READY_FOR_PAID_GPU=YES/NO；NO 时先在非付费/本地阶段补齐可完成的工作。

## Stage 8–9：评估与 Ablation

确认指标协议后再运行 evaluator。明确 reference 类型、配对、frame alignment、resize、normalization、aggregation、evaluator version、所需 frames/data 和输出格式。不要默认 PSNR/SSIM/LPIPS 是 generated-vs-original。reference 缺失时在 generation 前报告，标记相关指标 unavailable 和原因；禁止复制论文结果充当复现结果。

Ablation 只纳入已冻结范围内的项目。除研究因素外，尽量固定 backbone、checkpoint、环境、输入、seed、resolution、frames 和 steps。不同 ablation 不要另建整套大模型环境；只有代码结构确实要求独立环境时，先说明原因。Baseline 并非 ablation 的默认前提，只有 scope 或 reference protocol 要求才设置为 gate。

每个命令都要给出目的、执行命令、验证命令、预期输出、PASS/FAIL 和失败是否阻塞下一步；分阶段提供彼此有依赖的命令，避免一次让用户盲跑大量步骤。最终报告源代码和环境版本、冻结范围、bundle/output 路径、指标/ref protocol、resume 状态和唯一成功证据。

