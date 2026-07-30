# M5StickS3 开发技能

面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖可验证的板级 bring-up、固件编译上传、IR 红外收发、按钮、电源、LCD、ES8311 音频、NVS 和 BLE HID 陷阱。

## 安装

将本仓库的 GitHub URL 交给你的 AI agent（Codex / Claude Code / Cursor / OpenCode），让它读取 `skills/m5stack_sticks3.md` 并添加到 workspace 的 skill 发现链。

## 内容

- `skills/m5stack_sticks3.md` — canonical root skill，包含硬件引脚、跨框架边界、验收标准和已知陷阱
- `skills/m5stack_sticks3_esp_idf.md` — ESP-IDF 裸驱 LCD、ES8311、BLE HID、实时 WebSocket 和 token UI 边界
- `skills/m5stack_sticks3_m5unified.md` — Arduino/M5Unified 环境、显示、按钮、RMT IR 和 NVS

## 许可证

MIT
