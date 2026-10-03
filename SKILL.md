---
name: code-repro
description: Use when reproducing academic or machine-learning experiments, preparing research-model environments, running benchmarks or ablations, evaluating outputs, or reproducing code behavior and errors.
metadata:
  short-description: 可审计、可恢复、范围明确的代码与论文复现
---

# 代码复现

## 核心原则

先冻结用户范围和成功证据，再读核心代码并做只读预检；先完整审计问题和依赖，再一次性规划最小修复；验证并冻结环境后才运行正式实验。不要采用“遇到一个错误就安装一个包再重跑”的方式。

始终遵守用户明确指定的实验组、模型/backbone、checkpoint、样本、参数、指标、评估器、产物和排除项。不得擅自增加 baseline/original、额外模型、benchmark、指标或 ablation。若协议或科学有效性似乎要求用户未指定的项目，先说明原因并确认是否扩展范围；用户明确排除的项目是硬约束。安装、下载、进程启动都不是复现成功证据。

## 按需读取参考

- 论文、深度学习模型、benchmark、GPU 实验、metrics、evaluator 或 ablation：读取 [references/paper-reproduction.md](references/paper-reproduction.md)。
- 新环境预检、报错/日志审计、修复分组、环境冻结、长任务、run 事务或断点续跑：读取 [references/error-audit-and-recovery.md](references/error-audit-and-recovery.md)。

## 基本流程

1. **冻结任务范围。** 写清用户要求、明确排除项、目标证据和需要的最终产物。多模型或多个实验仅在用户要求或目标协议确实需要时纳入。缺少阻塞信息时，先做独立只读工作，再询问最小必要问题。
2. **读代码和官方规格。** 跟踪最短目标执行路径；检查官方 README、依赖清单、配置、子模块、模型组件和 evaluator 的真实 import/调用路径。先判断任务是训练、微调、纯推理还是 training-free 方法，再确定数据需求。
3. **先只读预检，再改环境。** 新模型、evaluator、benchmark 或 native/CUDA extension 首次接入时，先完成目标相关的只读审计。第一次审计不安装、升级或改配置。全新实例声明只限制无关盘点，不豁免目标所需审计。
4. **完整审计后最小修复。** 用户给出截图、traceback、日志或预检报告时，先提取所有可见错误和阻塞 warning，按共同根因及前置关系分组，再给可一次执行的修复方案。仅保留必须由前置修复揭示的后续检查为下一阶段。默认冻结已 PASS 的核心组件。
5. **验证、冻结并运行。** 环境 READY 后保存快照；只运行冻结范围内的任务。论文工作遵循参考中的阶段门禁、Evaluation Bundle、事务化和恢复规则。用户明确拒绝 smoke test 时，标记 N/A；仍做 load-only 和资源/依赖验证，并把首个正式样本作为事务化运行，失败即停。
6. **按证据报告。** 给出的命令应带目的、执行/验证方式、预期输出、PASS/FAIL 和失败是否阻断下一步。保存完整日志并报告实际证据、状态和限制。

## 实例模式与长任务

- 用户明确声明实例全新时，接受其声明；项目依赖、配置和目标资源按未准备处理，不扫描无关包、缓存、历史或进程。
- 已有实例只检查目标路径所需条件并复用兼容成果。
- 安装、下载、编译、推理和 evaluator 长任务必须将 stdout/stderr 持久写入日志，并使用 tmux、screen、调度器或平台支持的可恢复 job，避免依赖 SSH、Jupyter、浏览器或 IDE 会话持续连接。
- 结论只能来自可观察的成功证据。环境、模型加载、CUDA/native extension、数据、算法、evaluator 和结果错误须分层判断，不能用上层失败推断下层失败。

