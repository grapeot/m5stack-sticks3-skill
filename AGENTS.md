# AGENTS.md — M5StickS3 Skill

## 项目概述

面向 AI agent 的 M5StickS3 嵌入式开发技能文档。Public repo，中文。

## 项目结构

- `skills/m5stack_sticks3.md` — 完整技能文件
- `README.md` — 面向用户的安装说明
- `docs/` — 补充文档（如有）

## 维护要求

- 所有公开文件用 fake 占位符，不出现真实邮箱、API key、内部路径
- 新踩的坑追加到 `skills/m5stack_sticks3.md` 的"已知陷阱汇总"表格
- 更新后只对 tracked/public 文件跑隐私扫描，至少检查邮箱、`/Users/`、私网地址、`op://`、常见 token/key 和 PEM 私钥头；命中必须人工确认，不能把零命中等同于完成审查
- public repo 不使用指向 workspace 外部文件的 symlink

## 版本控制

只有用户明确要求时才 commit。
