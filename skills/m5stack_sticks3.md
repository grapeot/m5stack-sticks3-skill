# M5StickS3 开发技能

> 面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖硬件概览、引脚、按钮、电源、存储和已知陷阱。
>
> 子文件路由：
> - 自主刷写与测试闭环（框架无关）：`m5stack_sticks3_test_loop.md`（运行态自动刷写、刷后验收、测试结果契约、自动化边界）——做固件 bring-up 或持续迭代先读这份
> - ESP-IDF：`m5stack_sticks3_esp_idf.md`（项目配置、裸驱 LCD、ES8311 音频、BLE HID、实时 UI）
> - M5Unified / Arduino：`m5stack_sticks3_m5unified.md`（环境搭建、显示 API、IR RMT 与 NEC、按钮 API、NVS、省电）
> - iOS 中文输入：`docs/ios_chinese_input.md`（BLE HID Unicode 边界与 Custom Keyboard 架构）

## 元数据

- **类型**: BestPractice / API Guide
- **适用场景**: 在 M5StickS3 上开发 Arduino 或 ESP-IDF 固件，尤其是 IR、按钮、电源、ES8311 音频、RMT、NVS 和多机 ESP-NOW 链路
- **硬件**: M5StickS3 (ESP32-S3-PICO-1-N8R8, 8MB Flash, 8MB PSRAM)
- **创建日期**: 2026-07-29
- **最后验证**: 2026-09-27（双机 ESP-NOW 无加密链路、显示 HUD 分层刷新、download mode 刷写后留 ROM 坑；M5Stack Arduino core 3.3.8、arduino-cli 1.5.1）；2026-07-30（M5StickS3 SKU K150、ESP-IDF 5.5.5、`esp_codec_dev` 1.6.2、M5Stack Arduino core 3.3.8）

## 这个技能解决什么问题

M5StickS3 和前代 StickC/Plus/Plus2 在硬件上有大量不兼容之处。本技能记录在 StickS3 上从零开发固件时已经由官方 board-support 源码确认或在实机复现的板级约束，让后续 agent 避免把相近型号经验直接套到 StickS3。

### 边界

本技能覆盖板级 bring-up：开发环境、可供 agent 驱动的测试控制面、引脚、按钮、电源、LCD、IR、ES8311 音频、NVS，以及这些外设对实时网络/BLE 应用的约束。它不提供特定电视、云服务或语音产品的完整实现，也不把单一设备上的现象推广成某个消费电子产品系列的通用结论。

文中的结论分为三类：标注“实机验证”的内容已经在 SKU K150 上验证；引用 M5GFX/M5Unified/M5PM1 的内容来自官方 board-support 源码；其余代码片段是实现起点，必须经过下方验收，不应仅凭编译通过宣称硬件可用。

### 验收标准

一次新的 StickS3 固件 bring-up 至少满足适用项后才算完成：

| 能力 | 验收方式 |
|------|----------|
| 构建与刷机 | 从干净 build 目录为 `esp32s3`/`m5stack_sticks3` 编译成功；115200 baud 刷写完成且 image hash 校验通过 |
| 启动 | 串口确认芯片为 ESP32-S3、8MB Flash、8MB Octal PSRAM；无 boot loop |
| 自主迭代 | 完成一次无需物理按键的“运行态 → 刷写 → 读取新 build identity → 发送测试命令 → 收到关联的结构化结果 → 主机断言”；结果来自 StickS3 上的真实业务路径，不是 host stub |
| 按钮 | BtnA/G11、BtnB/G12 各产生一次预期事件；PWR/reset 只作为恢复路径验证 |
| LCD | 黑、白、纯红、纯绿、纯蓝色块位置和颜色正确；重复刷新无随机变色或 DMA 生命周期问题 |
| 音频 | 24kHz、16-bit、mono 采集 1 秒得到 48,000 bytes，PCM 非全零且峰值随环境声音变化；codec identity readback 单独不算成功 |
| IR（若使用） | 已关闭功放、开启 EXT_5V；已知 NEC 遥控器能稳定接收并通过反码/时序校验，回放由目标设备实际响应 |
| 网络/BLE（若使用） | 状态与日志不回显 secret；BLE HID 检查 report 返回值，并完成长于 95 字符的连续实机输入测试；同时使用 TLS 时，在 BLE 已连接状态完成一笔大于历史失败边界的上传 |

### 首要调试原则：查 board-support 源码，不猜

遇到引脚、panel flag、PMIC rail、按钮或 codec 初始化问题时，先查当前版本的 M5GFX/M5Unified 中 `board_M5StickS3` 分支，再参考通用 ST7789/ESP32 示例。M5Stack 的 board-support 源码直接编码了量产板所需的精确参数，优先级高于相近型号经验、社区帖子和肉眼试色。

例如 M5GFX 的 StickS3 分支明确给出：`Panel_ST7789`、135x240、offset `(52,40)`、`invert=true`、默认 RGB element order、40MHz write clock、CS=41、RST=21、MOSI=39、SCLK=40、DC=45。若一开始逐项照抄这些事实，就不需要在 RGB/BGR 和 inversion 之间反复猜测。只有源码与实机仍不一致时，才做最小化纯色/引脚实验，并把差异记录下来。

M5Unified 的 API 语义（`Mic_Class`/`Speaker_Class` 的缓冲与节拍、ES8311 寄存器写入时机、board 引脚映射）以本机安装的版本源码为准：sketchbook libraries 目录（macOS 通常 `~/Documents/Arduino/libraries/M5Unified`，`src/utility/` 下是各外设类）；arduino-cli 的 core 安装位置以 `arduino-cli config dump` 的 `data_dir` 为准（macOS 上可能是 `~/Library/Arduino15` 而非默认的 `~/.arduino15`，找不到 core 时先查这里）。写音频/多机协议代码前先读对应类源码确认语义，不要凭 API 名字猜。

## 硬件概览

### 芯片与外设

| 组件 | 型号 | 备注 |
|------|------|------|
| SoC | ESP32-S3-PICO-1-N8R8 | 双核 LX7, 240MHz |
| Flash | 8MB (embedded) | GD |
| PSRAM | 8MB Octal (embedded) | AP_3v3 |
| 显示 | ST7789P3, 135x240, 1.14" IPS | M5GFX 驱动 |
| IMU | BMI270 (0x68) | I2C |
| 音频 | ES8311 codec (0x18) + MEMS mic + AW8737 功放 + 8Ω speaker | I2S |
| IR | 发射器 GPIO46 + 接收器 GPIO42 | 38kHz, RMT 驱动 |
| 电源管理 | M5PM1 (0x6e) | I2C, 250mAh 电池 |
| USB | USB-C OTG, USB-Serial/JTAG | 原生 CDC, 非传统 UART 桥接 |
| Grove | HY2.0-4P | GND/5V/G9/G10 |
| Hat2 Bus | 16-pin | GPIO1-8, GPIO43, GPIO44 等 |

### 关键引脚

| 功能 | GPIO | 备注 |
|------|------|------|
| IR_TX | 46 | IR LED 发射 |
| IR_RX | 42 | IR 接收器, 必须 RMT 驱动 |
| BtnA | 11 | 正面 M5 主按键，wasPressed/wasHold/wasDoubleClicked |
| BtnB | 12 | 侧键 |
| PWR | M5PM1 | 电源键，硬件级，短按重启 |
| GPIO0 | 0 | ROM strap；没有独立物理 BOOT 键，不当作普通空闲 GPIO |
| LCD MOSI | 39 | |
| LCD SCK | 40 | |
| LCD RS | 45 | |
| LCD CS | 41 | |
| LCD RST | 21 | |
| LCD BL | 38 | 背光 PWM |
| I2C SCL | 48 | BMI270 + M5PM1 共用 |
| I2C SDA | 47 | |
| ES8311 MCLK | 18 | MCU → codec |
| ES8311 BCLK | 17 | MCU → codec |
| ES8311 LRCK | 15 | MCU → codec |
| I2S RX / ES8311 ASDOUT | 16 | codec → MCU，麦克风数据 |
| I2S TX / ES8311 DSDIN | 14 | MCU → codec，扬声器数据 |

## 开发环境

开发环境搭建、FQBN、编译上传等细节见 `m5stack_sticks3_m5unified.md`。

关键要点：M5Stack board package ≠ espressif esp32 core；FQBN 是 `m5stack:esp32:m5stack_sticks3`；8MB PSRAM 是 Octal/OPI。

## 按钮系统（核心踩坑区）

### 三个物理按钮

| 按钮 | GPIO / 来源 | 物理位置 | 固件可控制？ | API |
|------|-------------|----------|-------------|-----|
| BtnA | GPIO 11 | 正面 | ✅ 完全控制 | `M5.BtnA.wasPressed()` |
| BtnB | GPIO 12 | 侧面 | ✅ 完全控制 | `M5.BtnB.wasPressed()` |
| PWR/reset | M5PM1 PMIC | 侧面 | ❌ 不用于 UI | 短按复位、双击关机、长按进入下载模式 |

### BtnA / BtnB 事件 API

M5Unified 的 Button_Class 提供 `wasPressed`、`wasClicked`、`wasSingleClicked`、`wasDoubleClicked`、`wasHold`、`pressedFor` 等方法。完整 API 和单击/双击冲突解决方案见 `m5stack_sticks3_m5unified.md`。

### PWR/reset 电源键——硬件级，不用于 UI

PWR/reset 不是普通 GPIO。短按会触发硬件复位，双击会关机，USB 已连接时长按可进入下载模式。固件不能把这些默认硬件动作当成可靠的应用输入。

**结论：不要用 PWR 键做 UI 操作。** 只用 BtnA 和 BtnB。如果需要选择/确认/返回，用短按和双击/长按组合。这个 skill 的默认安全边界是：不改写 M5PM1 对 PWR 的复位、关机和下载模式默认动作；它是设备从固件卡死中恢复的最后路径。

### 推荐按钮映射方案

两个按钮够用，用短按 + 双击 + 长按三层交互。推荐映射符合横屏握持习惯：

| 操作 | BtnA（正面） | BtnB（侧面） |
|------|-------------|-------------|
| 短按 | 选择/进入/确认/发送 | 切换/翻页 |
| 双击 | 返回/取消 | （备用） |
| 长按 | （备用） | （备用） |

侧面按钮翻页 + 正面按钮确认的设计符合横屏握持：手指自然放在侧面翻动，正面拇指确认。

### 长按与短按的冲突

`wasPressed()` 在按下瞬间就触发，而 `wasHold()` 在持续按下后才触发。如果同时检测两者，短按操作会在长按之前就触发。解决方法：

- 如果短按和长按做不同的事，用 `wasClicked()`（等释放后才判定）代替 `wasPressed()`，或者
- 只用 `wasPressed()`（短按）和 `wasDoubleClicked()`（返回），不用长按。这是最简洁的方案。

## 红外 IR 收发

IR 收发细节（RMT 初始化、NEC 协议、解码校验）见 `m5stack_sticks3_m5unified.md`。

关键约束：
- IR_TX=GPIO46, IR_RX=GPIO42，收发器在设备顶部
- 接收前必须关功放 + 开 EXT_5V
- 用 RMT 外设（不是 GPIO 轮询）
- 38kHz 载波

## 电源与启动

### 进入 Download Mode（刷固件模式）

StickS3 没有独立 BOOT 键。进入下载模式的官方流程：

1. 连接 USB 线
2. **按住侧边 PWR/reset 键不放**
3. 内部绿色 LED 开始闪烁 = 已进入下载模式
4. 松开 PWR/reset 键

此时 esptool / arduino-cli 可以稳定连接。

绿灯闪烁只作为进入 Download Mode 的指示，其他灯态不下应用层结论。

### 从 Download Mode 正常启动

esptool 刷完后会自动发 hard reset，设备应从 flash 启动。如果因电池供电导致 reset 无效：
- 短按一下 PWR/reset 键（单击 = 硬件复位，会从 flash 正常启动）
- 如果短按也没用，双击 PWR 关机，再单击开机

### 自主刷写与测试闭环

运行态直接刷写的稳定闭环、刷后验收（为什么端口存在不算证据、卡 ROM 的非物理恢复）、刷后约 60 秒 sleep 窗口、测试结果契约（build_id / request_id / READY-RESULT 协议）、host runner 与自动化边界，见 `m5stack_sticks3_test_loop.md`。物理按键是恢复手段，不是正常迭代步骤。

### 电池与电源保持

StickS3 内置 250mAh 电池。**拔插 USB 不会强制重启设备**（电池持续供电）。

StickS3 不需要 MCU GPIO4 HOLD 来维持主电源。M5PM1 自身管理主电源保持，M5Unified 的 `power_hold` pin 表也不包含 StickS3。这里不要与音频轨的 M5PM1 `LDO_HOLD` 位混淆：后者仍需在纯 ESP-IDF 音频初始化中设置。如果需要软件关机，通过 M5PM1 I2C 命令实现，而不是拉低 GPIO4。

`M5.Power.getBatteryLevel()` 是**即时电池电压 → 百分比的映射（M5PM1），不是库仑计**：USB 供电（充电维持）时读数偏高，改电池带载运行（屏 + WiFi + 音频）后电压 sag，读数明显下降。实例：USB 下稳定 42-47%，拔电后立刻 10%——这是"电池快没电/内阻大"的真实反映，不是读取错误，不要当 bug 修。

### ES8311 与音频

ES8311 和 MEMS mic 由 `3V3_L3B_AU` 供电，不依赖 EXT_5V。ESP-IDF 裸驱初始化细节（M5PM1 LDO、`esp_codec_dev` 配置、验收方法）见 `m5stack_sticks3_esp_idf.md`；Arduino/M5Unified 的音频 API、默认配置、无声排查顺序见 `m5stack_sticks3_m5unified.md`。

### EXT_5V 输出

```cpp
M5.Power.setExtOutput(true, m5::ext_none);
```

该轨用于 Grove、Hat、IR，以及 **AW8737 喇叭功放**：`output_power = false` 时功放断电，喇叭完全无声（麦克风不受影响，它在 3V3 音频轨上）——音频场景保持默认 true。该轨不给 ES8311 或 MEMS mic 供电。

## 显示

显示 API（`drawCenterString`、datum、字体、竖屏坐标）见 `m5stack_sticks3_m5unified.md`。

ESP-IDF 裸驱 LCD（RGB565、byte endian、DMA 生命周期、panel inversion）见 `m5stack_sticks3_esp_idf.md`。

## 存储（NVS / Preferences）

NVS 用法见 `m5stack_sticks3_m5unified.md`。

## BLE HID

BLE HID 约束（NimBLE、iOS 配对、report 节奏）见 `m5stack_sticks3_esp_idf.md`。

标准 HID keyboard report 传输键位，不保证直接输入中文或任意 Unicode。iOS 对 HID Unicode Page、`\uXXXX` 键盘扩展替换方案的边界，以及推荐的 UTF-8 + Custom Keyboard 架构见 [`docs/ios_chinese_input.md`](../docs/ios_chinese_input.md)。

## ESP-NOW（多机无加密最小链路）

双机同步、玩具级遥控等不需要认证/加密的场景，在 Arduino core 3.3.8（IDF 5.x）上已验证的最小配方：

```cpp
WiFi.mode(WIFI_STA);   // 不关联 AP
esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);  // 两端固定同一信道
esp_now_init();

esp_now_peer_info_t peer = {};
memcpy(peer.peer_addr, peer_mac, 6);
peer.ifidx = WIFI_IF_STA;
peer.channel = 1;
peer.encrypt = false;
esp_now_add_peer(&peer);
esp_now_register_recv_cb(recv_cb);
```

- 收包回调签名以 IDF 5.x 为准：`void recv_cb(const esp_now_recv_info_t *info, const uint8_t *data, int len)`；IDF 4.x 是 `(const uint8_t *mac, const uint8_t *data, int len)`，旧签名直接编不过。
- 回调只做 memcpy 到局部 + 置 volatile 标志，状态机处理放主循环，与 IR RMT 回调的约束一致。
- `esp_now_add_peer` 对加密单播是必需项（driver 需要查 PMK/LMK，缺失时 `esp_now_send` 显式报错）；无加密单播不注册 peer 通常也能直接发到目标 MAC。配方里显式注册是推荐做法，省略前请先实测对端可达性。
- 两端信道不一致时帧被静默丢弃：没有 disconnect 事件、没有错误日志。排障第一步是两端各自 `esp_wifi_get_channel()` 核对。
- **禁止跨设备传绝对 `millis()`**：每台设备的 `millis()` 以自己的开机为基准，两台之间的差是任意的（实测可达数十秒），接收端拿对端时间戳与本地相减会无符号下溢，时序/倒计时卡死。只传周期/相位内的相对时间，接收端用本地收包时刻锚定。
- `esp_now_send()` 的返回值只表示提交给 driver，不代表对端收到；需要可靠性时应用层自己加序列号/心跳，或升级到加密单播 + 应用层认证的重型方案。
- **高频数据（~50 pkt/s 音频帧等）不要在回调里只置标志**：单槽标志 + 主循环处理会在丢包率放大到不可用（loop 周期 ≥ 包间隔时旧包未处理就被新包覆盖）。回调里直接写 SPSC 环形缓冲（单生产者 = ESP-NOW 任务，单消费者 = 专用播放/处理任务，head/tail 索引无锁，满了覆盖最旧 + 计数）+ 序列号；"memcpy + 置标志"模式只留给低频控制消息。
- `esp_now_recv_info_t.rx_ctrl` 是**指针**（`wifi_pkt_rx_ctrl_t *`）：RSSI 取 `info->rx_ctrl->rssi`，点访问直接编译错误。
- **周期包携带状态副本时，事件路径必须在两端都真的被处理**：否则周期包里的 stale 副本会按周期覆盖对端的新鲜本地状态（实例：1Hz 心跳携带按钮态，但收端没处理按钮事件包 → 本地 hold 状态每秒被 stale 0 覆盖一次，屏幕每秒闪一帧默认画面）。协议评审时逐条检查"每条事件路径在两个角色上都落地"。

## 入门陷阱速查

新 agent 首次开发最容易撞上、撞上一次代价高的坑。细节只在归属章节出现一次，这里只做索引。

| # | 陷阱 | 指针 |
|---|------|------|
| 1 | 用 espressif 官方 esp32 core 找不到 StickS3 板型 | m5unified §开发环境搭建 |
| 2 | FQBN 写成 `m5sticks3` | m5unified §FQBN |
| 3 | PSRAM 配成 Quad/QSPI → boot loop | esp_idf §项目配置 |
| 4 | 用 PWR 键做 UI → 短按就是硬件复位 | 本文 §按钮系统 |
| 5 | 没有 BOOT 键，进 download mode 靠长按 PWR | 本文 §电源与启动 |
| 6 | 每次刷写都要求用户长按 PWR/reset | test_loop §自主刷写闭环 |
| 7 | 刷后设备停在 ROM / 端口存在当应用在跑 | test_loop §刷后验收 |
| 8 | 把 build/flash 成功当功能验收 | test_loop §测试结果契约 |
| 9 | 喇叭无声先怀疑硬件或音量 | m5unified §无声排查顺序 |
| 10 | `output_power=false` 饿死功放，喇叭完全无声 | m5unified §音量与实测响度 |
| 11 | IR 接收前不关功放、不开 EXT_5V | m5unified §IR |
| 12 | 跨设备传绝对 `millis()` 时间戳 | 本文 §ESP-NOW |

## 参考资源

- M5Stack 官方文档: https://docs.m5stack.com/en/core/StickS3
- M5Stack IR NEC 例程: https://docs.m5stack.com/en/arduino/m5sticks3/ir_nec
- M5Unified GitHub: https://github.com/m5stack/M5Unified
- M5GFX GitHub（查 `board_M5StickS3`）: https://github.com/m5stack/M5GFX
- M5PM1 按键与电源控制: https://github.com/m5stack/M5PM1

## 安装方式

将本 skill 的 GitHub URL 交给 Codex / Claude Code / Cursor / OpenCode 等 AI agent，让它：

1. 读取目标 workspace 的 `AGENTS.md` 或 `CLAUDE.md`
2. 跟随路由文件（如 `WORKSPACE.md`）
3. 将本 skill 添加到 workspace 的 skill 发现链中
4. 如果 workspace 有 `rules/skills/INDEX.md` 或 `skills/INDEX.md`，更新索引
