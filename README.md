# Code Reproduction Skill

一个帮助 Codex **真正跑通指定代码或稳定复现目标行为** 的通用 Skill，适用于论文实现、开源仓库、服务、脚本和缺陷复现。

An adaptable Codex skill for reproducing actual code behavior with a minimal, observable path. For explicitly fresh server instances, it skips broad inventories and performs one need-based preflight before installing the minimum dependencies and running a smoke test.

## 核心流程

1. 先定义唯一成功证据，明确如何观察和判定目标行为。
2. 阅读核心代码，追踪实现目标行为的最短执行路径。
3. 列出该路径必需的依赖、输入、配置和最小模型文件。
4. 普通实例按清单检查已有环境；明确声明全新实例时，只做一次必要的基础预检。
5. 只补缺失或有证据损坏、不兼容的项目。
6. 必需项就绪后立刻运行最小 smoke test。
7. 报告实际证据；失败时读取错误、针对性修复并重新验证。

完整操作规则见 [SKILL.md](SKILL.md)。依赖、输入和判据由每个任务的目标行为与核心代码决定；Skill 不携带项目源码、权重或数据集。

## 安装与使用

将本文件夹放进 Codex 的 skills 目录：

- Windows：`%USERPROFILE%\.codex\skills\code-repro`
- macOS/Linux：`~/.codex/skills/code-repro`（若设置了 `CODEX_HOME`，使用其下的 `skills/code-repro`）

随后可在相关任务中指定 `$code-repro`，也可由 Codex 自动选择。

## 全新实例快速启动

如果当前服务器/实例是全新的，且项目还没有配置，可以在任务中说明：

> 这是全新实例：项目依赖、配置、数据和模型都未准备。请按 `code-repro` 的全新实例快速启动模式执行。

Skill 会跳过用全量盘点来验证实例是否全新，只检查最小执行路径需要的主机条件，然后准备依赖并尽早运行 smoke test。遇到具体错误时再检查相关环境项。

## 复现证据

复现报告围绕预先定义的唯一成功证据，附上源码版本、环境和输入、目标路径日志/响应/产物、smoke test 命令与退出状态。依赖安装和资源就绪属于准备状态，不单独证明目标已复现。

## License

本项目采用 [MIT License](LICENSE)。





