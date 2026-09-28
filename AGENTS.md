# AGENTS.md — M5StickS3 Skill

## 项目概述

面向 AI agent 的 M5StickS3 嵌入式开发技能文档。Public repo，中文。

## 项目结构

- `skills/m5stack_sticks3.md` — 主 skill：入口路由 + 板级事实 + 验收标准 + 入门陷阱速查
- `skills/m5stack_sticks3_test_loop.md` — 自主刷写与测试闭环（框架无关）
- `skills/m5stack_sticks3_esp_idf.md` — ESP-IDF 框架子文件
- `skills/m5stack_sticks3_m5unified.md` — M5Unified / Arduino 框架子文件
- `README.md` — 面向用户的安装说明
- `docs/` — 补充文档（如有）

## 维护要求

- 所有公开文件用 fake 占位符，不出现真实邮箱、API key、内部路径
- 新踩的坑的落点：每个坑只写进它归属的章节（框架相关 → 对应框架子文件；刷写/测试闭环 → `m5stack_sticks3_test_loop.md`；物理/板级/电源 → 主文件）。归属章节已覆盖的坑不再单独立条目
- 只有跨框架且首次撞上代价高的坑，才允许在主文件"入门陷阱速查"索引加一行；索引硬上限 12 行，加一行必须移出一行；索引行只写一句话根因 + 指针，细节只在归属章节
- 规模护栏：主文件 ≤280 行 / ≤18KB；子文件单节超过 40 行且属于单一框架时，拆进对应子文件
- 更新后只对 tracked/public 文件跑隐私扫描，至少检查邮箱、`/Users/`、私网地址、`op://`、常见 token/key 和 PEM 私钥头；命中必须人工确认，不能把零命中等同于完成审查
- public repo 不使用指向 workspace 外部文件的 symlink

## 版本控制

只有用户明确要求时才 commit。
