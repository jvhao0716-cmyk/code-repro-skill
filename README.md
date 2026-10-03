# Code Reproduction Skill

一个用于论文、开源仓库、模型、benchmark、evaluator 和错误复现的 Codex Skill。先冻结用户范围和指标，再只读审计、完整归因错误、集中做最小修复、冻结环境，最后生成可恢复的结果并独立评估。

## 核心流程

scope freeze → metric-first planning → read-only preflight → full error audit → batched minimal repair → final validation/freeze → transactional resumable runs → evaluation → evidence report

用户明确排除的 baseline、模型、指标或 ablation 不会被擅自加入。Smoke test 默认用于降低风险，但用户明确拒绝时可标记 N/A；仍验证依赖、checkpoint、数据和模型加载，并安全运行第一个正式样本。Training-free 方法不自动要求训练集。

## 文件

- [SKILL.md](SKILL.md)：触发范围、总原则和流程路由。
- [论文与机器学习实验门禁](references/paper-reproduction.md)：范围冻结、只读预检、指标优先、可选 smoke test、生成 bundle、评测和 ablation。
- [错误审计与恢复](references/error-audit-and-recovery.md)：根因分组、最小修复、环境冻结、日志、断连保护、失败隔离和断点续跑。

本 Skill 不携带项目源码、模型权重或数据集。

## 安装与使用

将本文件夹放进 Codex 的 skills 目录：

- Windows：%USERPROFILE%\.codex\skills\code-repro
- macOS/Linux：~/.codex/skills/code-repro（若设置了 CODEX_HOME，使用其下的 skills/code-repro）

随后可在相关任务中指定 $code-repro，也可由 Codex 自动选择。

## 全新实例

可在任务中说明：“这是全新实例：项目依赖、配置、数据和模型都未准备。请按 code-repro 的全新实例模式执行。”

Skill 会接受该声明，不盘点无关包、缓存、历史或进程；仍会对目标路径做只读需求审计。明确排除的工作保持排除。

## License

本项目采用 [MIT License](LICENSE)。

