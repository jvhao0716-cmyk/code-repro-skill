# Error Audit 与恢复策略

本参考用于新模型/evaluator/extension 首次接入、环境配置、用户返回 screenshot/terminal output/traceback/log/preflight report，以及长任务失败后的恢复。核心规则：先只读收集并完整审计，再按根因制定最小修复；不采用“看到第一个错误就 pip install 一个包”的逐条试错。

## 1. 只读采集与完整 Error Audit

首次接入新模型、world model、evaluator、CUDA extension 或 benchmark 时，首次 Preflight 必须只读：查看规格、路径、版本、资源和日志；不得安装、升级、下载大文件、修改 environment/config 或启动昂贵实验。

用户给出报告后，先从已提供的全部材料中提取可观察问题，不要只处理第一条 traceback。建立 ERROR AUDIT，至少记录：

| 字段 | 内容 |
|---|---|
| observation | 原始错误/warning 文本、来源文件/截图/行号 |
| category | 系统资源、环境依赖、CUDA/native extension、模型/checkpoint、dataset/input、权限/路径、算法代码、evaluator/metric |
| root cause | 已证实根因或待验证假设；不要把症状写成根因 |
| dependency/blocker | 该问题依赖哪些前置修复，遮蔽哪些下游检查 |
| impact | 阻止哪个阶段/样本，是否可能破坏已 PASS 组件 |
| evidence/status | 已确认、推断、未知、被前置错误遮蔽 |
| action | 最小处理方式、验证信号及失败后停止点 |

合并共同根因，而不是按错误行数分类。例如 Torch/CUDA 不兼容可能同时造成 flash-attn、xformers 和 custom op 加载失败。不要把一个根因包装成多个独立安装任务。Warning 只有与目标路径和 gate 有关时才阻塞；说明忽略依据。

如果 screenshot 或日志完整展示多个问题，必须一次性提取。只有关键信息被截断或被前置失败真实遮蔽时，才向用户请求最小补充信息；说明哪些问题已确认、哪些可先一起修、哪些要在前置问题解决后才能观察。不要要求用户逐条截图来揭示本次材料中已经可见的错误。

## 2. 根因分组、阻塞图与批量修复

先画清修复顺序：

    confirmed root cause -> blocked checks -> downstream verification

再把修复分为：

1. **可并行/互不阻塞：**合并到同一 repair plan/script 和一次执行。
2. **有依赖：**先修复真正阻塞后续观测的前置项；明确告诉用户后续状态当前未知，以及完成前置步骤后要检查什么。
3. **证据不足：**保持未知；不得保证一条命令肯定修完。

修复计划应列出每个根因、证据、精确修改、影响的组件、保留不动的 PASS 组件、执行顺序和统一验收命令。能一次安全解决的缺包、CUDA_HOME、路径、缓存目录和权限问题，尽量一次性处理后统一验收；不能一次解决时解释具体阻塞关系。

## 3. 最小修改与已通过环境冻结

- PASS 的 Torch、CUDA、NumPy、关键模型库和 native backend 默认冻结。没有直接证据时不升级、不重装、不替换它们。
- 优先锁定满足官方规格的精确版本，并用 constraints/lock 保持关键包一致。
- 只在已确认安全、且安装器 otherwise 会改动已通过依赖时考虑 no-deps 安装；之后必须单独验证被省略的依赖闭包。
- 禁止无证据的 pip install -U、conda update、重装 PyTorch、升级 NumPy/Transformers 或重下整套模型。
- 缺少一个 Python 包时不得清空 cache 或重新下载 checkpoint。
- 每次 repair 后重新运行受影响的依赖/版本、导入、CUDA/backend 或文件检查，并生成新的环境快照。

如果一层失败令下游状态不可观测（例如 torch 本身无法 import，custom CUDA backend 的状态就尚未知），不要声称下游通过或失败；先记录为 BLOCKED，再在前置层通过后检查。

## 4. Environment freeze 与生成/评估隔离

正式验证通过后保存并引用：

- pip freeze、conda list（适用时）、environment.yml/lock。
- Python、Torch、Torch CUDA、driver、关键依赖版本和必要的 GPU 信息。
- Git commit、checkpoint/model identifier、manifest/config。

保存为 env_before.txt、env_final.txt 或等价 artifact；变更环境后重写 final snapshot。记录时避免 token、密码及无关个人环境变量。

生成模型与 evaluator/benchmark 如依赖不兼容或需要额外的大型依赖，优先分成 model_gen_env 与 evaluator_env。不得为了补 GroundingDINO、DROID、CLIP、pyIQA 等评测组件而破坏已通过的生成环境。ENV_READY=YES 后原则上只运行；新的运行错误应先停止受影响 run、保存日志并重新审计，然后有计划地修复、验证和更新快照，不在长任务中边跑边热修环境。

## 5. 持久日志与终端断连保护

安装、编译、下载、inference 和 evaluation 的 stdout/stderr 都要完整持久化到磁盘日志，并记录命令、退出码、阶段、时间和 artifact 路径。可使用 shell 的日志 tee 等价方式，但要确保日志捕获不会吞掉真实退出码（例如在适用 shell 中启用 pipefail）。

正式长任务不能绑定普通 SSH/Jupyter/VS Code/browser 终端。优先使用 tmux、screen、systemd-run、scheduler 或平台 job system，使用户断开会话后任务继续。若用 tmux，明确 session 名、磁盘日志路径和 detach/reattach 方法。启动前说明预期状态，完成后检查进程、退出码与完整日志。

用户只给截图时，尽量从图中一次性提取所有错误；要求完整日志时说明截断具体挡住了哪项判断，不要只说“再发一下最后几行”。

## 6. Per-sample transactional run、DONE/FAILED 与恢复

每个 sample/run 写入独立临时目录，初始状态 PENDING。成功判定至少包括：

- exit_code == 0。
- 目标产物存在、大小合理、格式/结构/预期帧数正确。
- 必需 input/config/metadata/runtime log 完整。
- 方法需要的算法机制证据存在。

满足后先持久化全部 bundle，再原子标记 DONE 或 SUCCESS。否则标记 FAILED，保存退出码、失败 stage、错误日志和可恢复信息；失败项绝不进入正式 aggregate metrics。文件名存在本身不能代表 DONE。

恢复前检查每个 sample 的 status、产物和 metadata：只跳过仍满足完整成功判据的 DONE，重跑 FAILED 或不完整项。N 个样本逐个提交；若 1、2 成功而 3 失败，重启后跳过 1、2，只重试 3。不得把全部样本积累到最后才统一保存。

生成与评估分离：bundle 应含后续 evaluator 真正需要的 output、input/reference identifiers、generation/algorithm config、runtime metrics、environment reference、logs 和 status。根据项目需要添加 frames、prompt、camera metadata；不需要的内容标 N/A，不制造无用占位文件。Generation 完成后 evaluator 应能重复运行而无需重做模型生成。

失败初始化耗时、加载耗时或总 wall time 不等同于 inference latency。runtime metrics 必须在有效推理区间实时采集；失败 run 不记录为成功指标。

## 7. 付费 GPU 成本 gate

GPU VM 启动前，尽量在本地或非付费阶段完成源码追踪、scope/manifest、依赖规格审计、样本选择、metric/reference protocol、evaluator 计划、output schema、日志/resume 设计和脚本检查。昂贵机器上只执行必须依赖该 GPU 的真实验证或正式运行。

启动付费 GPU 前输出 READY_FOR_PAID_GPU=YES/NO 和依据。任何可在本地完成的前置项未完成时为 NO；NO 时先完成该工作，不用开机做临时实验设计。

## 8. 错误分类与用户命令契约

至少区分：

1. 系统/资源：RAM、VRAM、disk、driver、OS、网络。
2. 环境/依赖：Python、package version、PATH 或环境变量。
3. CUDA/native extension：ABI、编译产物、动态库或 backend 调用。
4. 模型/checkpoint：配置、权重、tokenizer、组件或加载。
5. Dataset/input：路径、结构、样本、prompt、metadata/reference。
6. 算法实现：在环境、模型和输入均通过后，核心目标路径仍不正确。
7. Evaluator/metric：工具、协议、reference 或聚合错误。
8. 实验结果：有效运行成功但输出/统计不满足预先定义判据。

不得从某一层失败推断更下游结论。例如 CUDA extension 未加载不能据此判定算法错误。

每条发给用户的命令或修复脚本都说明：

- 目的与修改范围；哪些 PASS 组件会保持不变。
- 执行命令及统一验证命令。
- 预期输出与 PASS 条件。
- FAIL 的停止点及是否阻止下一阶段。

不要一次发送几十条相互依赖的命令。先提供一个阶段内可执行、可验收的命令组；被前置错误遮蔽的项目留到明确的下一阶段。

