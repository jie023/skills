---
name: "git-commit-helper"
description: "Use when the user explicitly asks to inspect Git changes, stage files, create commits, write commit messages, or perform Git operations."
---
# Git 提交助手

## 使用场景

- 用户明确要求查看 Git 变更、暂存文件、提交代码或生成提交信息
- 用户要求按规范整理 commit message
- 用户要求执行与 `git add`、`git commit`、`git push` 相关的操作

## 核心原则

- 无明确要求时，不执行或触碰任何 Git 操作，包括 `git status` 和 `git diff`。
- 不使用破坏性 Git 命令，除非用户明确要求并再次确认。
- 不执行 `git push --force`，除非用户明确要求并完成高风险确认。
- 一个 commit 只做一件事，不把无关文件混入提交。

## 提交信息规范

格式：

```text
<type>(<scope>): <subject>
```

常用 type：

| type | 用途 |
| ---- | ---- |
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `docs` | 文档 |
| `refactor` | 重构 |
| `test` | 测试 |
| `chore` | 构建或配置 |

禁止使用 `update`、`修改`、`提交代码` 等笼统描述。

## 多平台执行方式

- Claude：只有在用户明确要求 Git 操作时，才使用 Bash 或 Git 工具。
- Codex：只有在用户明确要求 Git 操作时，才运行 `git` 命令；优先使用非交互式命令。
- Trae：只有在用户明确要求 Git 操作时，才使用平台可用 Git 能力。

## 执行流程

1. 确认用户已经明确要求 Git 操作。
2. 只检查完成该 Git 请求所需的最小变更范围。
3. 提交前确认将要纳入的文件与用户目标一致。
4. 生成符合规范的 commit message。
5. 执行提交后报告提交哈希、提交信息和纳入范围。

## 高风险操作

以下操作必须单独确认：

- `git push --force`
- `git reset`
- `git checkout` 覆盖本地文件
- 删除分支、改写历史、清理未跟踪文件

## 禁止事项

- 禁止无请求执行 `git status`、`git diff`、`git add`、`git commit`、`git push`。
- 禁止用交互式 Git 流程替代明确的非交互式命令。
- 禁止提交与当前任务无关的文件。
- 禁止为了让工作区“干净”而还原用户未要求还原的改动。
