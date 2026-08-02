# M5StickS3 开发技能

> 面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖硬件概览、引脚、按钮、电源、存储和已知陷阱。
>
> 框架特定内容在子文件：
> - ESP-IDF：`m5stack_sticks3_esp_idf.md`（裸驱 LCD、ES8311 音频、BLE HID、实时 WebSocket）
> - M5Unified / Arduino：`m5stack_sticks3_m5unified.md`（环境搭建、显示 API、IR RMT、按钮 API、NVS）

## 元数据

- **类型**: BestPractice / API Guide
- **适用场景**: 在 M5StickS3 上开发 Arduino 或 ESP-IDF 固件，尤其是 IR、按钮、电源、ES8311 音频、RMT 和 NVS
- **硬件**: M5StickS3 (ESP32-S3-PICO-1-N8R8, 8MB Flash, 8MB PSRAM)
- **创建日期**: 2026-07-29
- **最后验证**: 2026-07-30（M5StickS3 SKU K150、ESP-IDF 5.5.5、`esp_codec_dev` 1.6.2、M5Stack Arduino core 3.3.8）

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

### 从 Download Mode 正常启动

esptool 刷完后会自动发 hard reset，设备应从 flash 启动。如果因电池供电导致 reset 无效：
- 短按一下 PWR/reset 键（单击 = 硬件复位，会从 flash 正常启动）
- 如果短按也没用，双击 PWR 关机，再单击开机

### 优先建立自主刷写闭环

StickS3 的原生 USB Serial/JTAG 支持从正常运行态自动进入 ROM download mode。只要应用没有关闭或重配 USB、设备没有进入会让 USB 失效的睡眠状态、主机仍能看到 CDC 端口，就应直接执行：

```bash
idf.py -p /dev/cu.usbmodemXXX -b 115200 flash
```

实机验证的稳定闭环是：

```text
正常运行态
  → esptool 通过 USB Serial/JTAG 自动进入 download mode
  → 115200 baud 写入并校验 image hash
  → --after=hard_reset
  → 新固件正常运行
  → 通过网络 status/health 接口确认版本或测试结果
```

这应当是 agent 的默认开发循环。**不要每次刷写前都让用户长按 PWR/reset 进入下载模式。** 物理按键是恢复手段，不是正常迭代步骤。第一次 bring-up 或固件已经破坏 USB 通路时，可以请用户做一次物理恢复；设备回到正常运行态后，应立即验证并保护上述自主闭环。

刷写后的验收也要避免改变被测状态。USB CDC/monitor 的打开、关闭和 DTR/RTS 操作可能再次触发 `USB_UART_CHIP_RESET`；设备本来停在 ROM download mode 时，打开 monitor 只会再次看到 `DOWNLOAD(USB/UART0)`，不能据此证明刚才的应用从未启动。优先用网络 health/status、LED 测试模式或其他独立信号确认运行。需要串口日志时，再明确把“观察应用”和“触发 reset”分开设计。

#### 何时才请求用户介入

先检查端口、当前网络状态和 esptool 连接结果，再决定是否需要物理操作：

| 当前状态 | Agent 行为 |
|----------|------------|
| 应用正常运行，USB CDC 端口存在 | 直接自动刷写；不要请求按键 |
| 刷写成功且网络 health/status 恢复 | 自主闭环成立，继续迭代 |
| 设备已停在物理触发的 ROM download mode，刷后仍未运行 | 请求短按一次 PWR/reset；回到运行态后重新验证自动刷写 |
| CDC 端口消失，但设备仍在网络上 | 检查 light/deep sleep、USB pin/console 配置；优先通过网络命令恢复或重启 |
| CDC 和网络都不可达、固件 boot loop、USB 被重配 | 明确请求长按 PWR/reset 至绿灯闪烁，只做一次恢复刷写 |
| 电源状态不明或短按无效 | 请求双击关机再单击开机，随后恢复自主闭环 |

请求用户帮助时，要说明**为什么机器已经越过自主能力边界、需要做哪个动作、预期把设备送到什么状态**。用户完成后 agent 应立即继续，不把后续机器可完成的步骤再交还给用户。

### 从自主刷写升级为自主实验

刷写成功只证明 image 可以写入，不能证明目标功能成立。高效的 agent 开发循环应形成下面的闭环：

```text
修改源码 → clean build → flash/hash verify → 读取新 build identity
        → 机器触发一个测试 → 设备执行真实业务路径
        → 返回结构化结果和原始测量 → 主机断言 → 保留失败证据或继续迭代
```

固件应暴露一个最小测试控制面，可以使用 USB serial、网络或 BLE，但必须满足以下结果契约：

- 启动后输出唯一的 `build_id` 或 git SHA，主机据此拒绝旧固件、旧端口和缓存响应。
- 每条命令带 `request_id`；每次测试只有一个可关联的终态 `pass`、`fail` 或 `error`，并有明确 timeout。
- 结果至少包含 `build_id`、`request_id`、测试名、状态、耗时、关键原始测量和错误细节。不要只输出自然语言 `OK`。
- 测试入口调用产品固件正在使用的 preprocessing、driver、inference、storage 或 network 路径；只在边界注入可控输入，不另写一套永远成功的测试实现。
- 固定输入记录长度/hash，随机流程记录 seed。主机应先验证输入完整性，再判断设备输出。
- 日志明确标记 `simulator`、`host` 或 `device`。只有结果确实由 StickS3 执行时才能标记 `device`。
- 未认证控制面不接收或回显 secret；测试结果中的地址、token 和用户数据必须脱敏。

协议不必复杂。line-delimited JSON 或稳定的 key-value 行就足够，例如：

```text
READY build_id=abc123 target=esp32s3
RESULT build_id=abc123 request_id=17 test=audio_capture status=pass elapsed_ms=1032 bytes=48000 peak=812
```

主机 runner 负责发现动态端口、等待匹配的 `READY`、发送命令、验证 `request_id` 和输入 hash、解析结果并以非零 exit code 表示断言失败。完整原始串口记录应作为失败证据保留；CLI 不要把 timeout、设备重启或底层错误压缩成笼统的 `test failed`。

当完整应用无法解释故障时，优先做最小 measurement firmware，而不是继续猜。它只保留当前被测链路及其供电、时钟和输入，输出可量化的原始结果；确认基线后再逐项加回显示、网络、BLE、存储和睡眠。每次只改变一个主要变量，并保留最后一个通过的 firmware hash，便于二分回归和恢复。

### 自动化边界

测试控制面可以注入按钮对应的逻辑事件、固定 PCM、网络 payload 或其他确定输入，从而减少重复人工操作，但不能把注入结果冒充物理验收。以下边界仍需要真实设备或用户动作：

- 首次接线、USB 控制面完全失联后的恢复，以及移动、遮挡或对准设备。
- 按钮电气/机械行为、LCD 实际颜色与布局、扬声器声音、麦克风环境响应、IR/RF 目标响应和真实功耗。
- deep sleep、USB 重配置或 GPIO19/GPIO20 用途会主动切断控制面时的最终场景验收。

进入会切断控制面的状态前，先输出最终结构化结果并等待传输完成，同时保留定时唤醒、网络恢复命令或已知可工作的恢复 image。Agent 应把人工动作压缩到这些物理边界，而不是把整轮构建、刷写和日志判断交给用户。

### 电池与电源保持

StickS3 内置 250mAh 电池。**拔插 USB 不会强制重启设备**（电池持续供电）。

StickS3 不需要 MCU GPIO4 HOLD 来维持主电源。M5PM1 自身管理主电源保持，M5Unified 的 `power_hold` pin 表也不包含 StickS3。这里不要与音频轨的 M5PM1 `LDO_HOLD` 位混淆：后者仍需在纯 ESP-IDF 音频初始化中设置。如果需要软件关机，通过 M5PM1 I2C 命令实现，而不是拉低 GPIO4。

### ES8311 与音频

ES8311 和 MEMS mic 由 `3V3_L3B_AU` 供电，不依赖 EXT_5V。ESP-IDF 裸驱初始化细节（M5PM1 LDO、`esp_codec_dev` 配置、验收方法）见 `m5stack_sticks3_esp_idf.md`。

Arduino/M5Unified 用户不需要手动初始化 ES8311——`M5.begin()` 会自动配置。如需关闭功放（IR 接收前），调 `M5.Speaker.end()`。

### EXT_5V 输出

```cpp
M5.Power.setExtOutput(true, m5::ext_none);
```

该轨用于 Grove、Hat 和 IR，不给 ES8311 或 MEMS mic 供电。

## 显示

显示 API（`drawCenterString`、datum、字体、竖屏坐标）见 `m5stack_sticks3_m5unified.md`。

ESP-IDF 裸驱 LCD（RGB565、byte endian、DMA 生命周期、panel inversion）见 `m5stack_sticks3_esp_idf.md`。

## 存储（NVS / Preferences）

NVS 用法见 `m5stack_sticks3_m5unified.md`。

## BLE HID

BLE HID 约束（NimBLE、iOS 配对、report 节奏）见 `m5stack_sticks3_esp_idf.md`。

标准 HID keyboard report 传输键位，不保证直接输入中文或任意 Unicode。iOS 对 HID Unicode Page、`\uXXXX` 键盘扩展替换方案的边界，以及推荐的 UTF-8 + Custom Keyboard 架构见 [`docs/ios_chinese_input.md`](../docs/ios_chinese_input.md)。

## 已知陷阱汇总

| 陷阱 | 表现 | 应对 |
|------|------|------|
| 用 espressif 官方 esp32 core | 找不到 StickS3 board 定义 | 用 `m5stack:esp32` board package |
| FQBN 写成 `m5sticks3` | 编译报 unknown board | 正确是 `m5stack_sticks3`（带前缀） |
| sketch 放在 `src/foo.ino` | `main file missing from sketch: src/src.ino` | 放在 `src/foo/foo.ino` |
| PlatformIO / ESP-IDF 把 PSRAM 配成 Quad/QSPI | PSRAM 初始化 panic、黑屏或 boot loop | StickS3 是 8MB Octal/OPI PSRAM；核对 `qio_opi` 或 IDF Octal PSRAM 配置 |
| 用 PWR 键做 UI | 短按触发硬件复位，设备重启 | 只用 BtnA 和 BtnB |
| IR RX 不关功放或未开 EXT_5V | 接收到噪声、无信号，或 IR TX/RX 完全不工作 | `cfg.internal_spk = false` + `M5.Speaker.end()` + `M5.Power.setExtOutput(true, m5::ext_none)` |
| 用旧版 IRremote 库 | 不兼容 ESP32-S3 RMT | 用 `driver/rmt_tx.h` / `driver/rmt_rx.h` |
| RMT RX callback 返回值一律写死 | 误报调度状态或漏掉唤醒 | 只写 volatile 标志时返回 false；用 FromISR API 唤醒高优先级 task 时返回 wake 标志 |
| NEC 解码用单一阈值 `space > 1000` | 噪声和异常 space 被接受 | 用窗口：0 = 300-1000us, 1 = 1200-2200us |
| NVS 多次更新相关 key | 断电时可能出现部分新值、部分旧值 | 用带版本和固定宽度字段的单 key blob，并校验长度 |
| 拔 USB 后设备不重启 | 电池供电，拔 USB 不断电 | 双击 PWR 关机后单击开机，或短按 PWR 复位 |
| 把 ES8311 供电误认成 BOOST 5V | ES8311 身份寄存器读取失败或 codec 未上电 | 打开 M5PM1 LDO_EN/LDO_HOLD，并将 GPIO2 `PYG2_L3B_EN` 输出高电平 |
| 手抄部分 ES8311 寄存器 | 身份 readback 正常但 PCM 严格全零 | 使用 `esp_codec_dev_open()` + `esp_codec_dev_set_in_gain()` 完成 open/enable/gain 状态机 |
| 误以为需要 GPIO4 HOLD | 不必要地占用 GPIO4，或误删音频 `LDO_HOLD` | 主电源不需要 GPIO4 HOLD；音频 L3B 仍需 M5PM1 `LDO_HOLD` |
| 进入下载模式用按 BOOT 插 USB | StickS3 没有 BOOT 键，操作无效 | 连 USB 后长按侧边 PWR/reset 直到绿灯闪 |
| 每次刷写前都要求用户长按 PWR/reset | 人为打断可自动化的开发循环，无法连续实验或二分回归 | 运行态且 CDC 端口存在时直接 `idf.py ... flash`；物理 download mode 只用于恢复 |
| 用重新打开 serial monitor 作为刷后唯一验收 | DTR/RTS 或 USB CDC open 再次触发 reset，观察动作改变设备状态 | 刷后先用网络 health/status 或独立信号验收；串口观察与 reset 控制分开 |
| 把 build/flash 成功当成功能验收 | image hash 正确，但新固件未启动、输入损坏或真实业务路径仍失败 | 校验新 `build_id`，再完成“命令 → device 结果 → host 断言”闭环 |
| 测试固件另写一套简化实现 | 测试全绿，产品中的 preprocessing、driver 或 postprocessing 仍有 bug | 复用产品路径，只在输入和输出边界增加控制面与 telemetry |
| 测试结果没有 build/request identity | 把旧端口、重启前日志或上一次响应误判为当前实验 | 每条 `READY`/`RESULT` 带 `build_id`，每个测试带唯一 `request_id` |
| 用 host/simulator 结果代替实机结果 | 本机算法通过，但芯片量化、内存、时序或外设路径失败 | 结果标记执行层；device 验收必须由 StickS3 运行并回传原始测量 |
| 结果发完前进入 deep sleep | 主机 timeout，无法区分测试失败、日志未 flush 和 USB 消失 | 发送终态并等待传输完成后再睡眠；保留定时唤醒或恢复 image |
| 把绿灯状态当成应用诊断 | 闪烁时误判崩溃，或从其他灯态推断应用正常 | 只把绿灯闪烁解释为 Download Mode；其他灯态不下结论 |
| 端口运行时消失 | 固件未启用 CDC、boot loop、低功耗或动态端口变化 | 先检查 CDC-on-boot 和串口日志；无法恢复时连接 USB，长按 PWR/reset 直到绿灯闪烁 |
| BLE HID 延时小于一个 FreeRTOS tick | 文字约 17 字符后截断，NimBLE 报 `Unable to fetch protocol_mode` | 检查 `CONFIG_FREERTOS_HZ`；100Hz 时 `pdMS_TO_TICKS(5)` 为 0。40 个 msys buffer 配合返回值检查和有界重试时，10ms 已通过连续 95 字符实机测试 |
| NimBLE host buffer 全放 internal RAM，同时运行 TLS | TLS connect 或大 body write 失败，`esp_http_client_write()` 可能返回 0 且 socket errno 仍为 0，看起来像 MTU、ACK 或 Wi-Fi 故障 | StickS3 有 8MB PSRAM 时优先 `CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_EXTERNAL=y`；在 BLE 已初始化/已连接状态用超过旧失败边界的 HTTPS request 做 A/B 验收 |
| 每个 BLE HID key 都发送 press + release | 打字速度只有必要 report 数量的一半 | 不同 key 可直接用下一份状态 report 替换，自动释放旧 key；相同连续 key 必须先发空 report，字符串结尾必须 release |
| 看到 BGR 配置后又手动交换 RGB565 红蓝位 | 纯蓝显示红褐色或紫色，palette 无法直觉调整 | StickS3 实机 framebuffer 使用标准 RGB565；先用纯色块单独验证 element order |
| 凭相近 ST7789 板型猜 panel 参数 | offset 对了但 inversion/order 错，反复调色仍不稳定 | 直接查 M5GFX `board_M5StickS3`：RGB order、`invert=true`、offset `(52,40)` |
| RGB565 framebuffer 未声明 little endian | 深蓝 `0x0009` 显示草绿，蓝色 `0x00DF` 显示黄色 | `esp_lcd_panel_dev_config_t.data_endian = LCD_RGB_DATA_ENDIAN_LITTLE` |
| `draw_bitmap` 返回后立刻复用或释放 buffer | 图形大致正确但颜色随机，深色 banner 变黄、浅色文字变红 | `on_color_trans_done` 发 semaphore；DMA 完成后才能改写栈/static buffer 或 `free()` heap buffer |
| StickS3 ST7789P3 未开启 color inversion | 白色显示黑色、深色显示浅色，怎么调 palette 都不对 | panel init 后调用 `esp_lcd_panel_invert_color(panel, true)`，再检查 BGR/endian |
| 音量条每个 audio chunk 多次清屏/填色 | meter 闪烁，按钮和录音链路变迟钝 | 音频先入队；静态 framebuffer 一次合成、一次 DMA，LCD 限制约 10fps |
| drawHeader 不设字体 | 继承前一个屏幕的字体，显示异常 | 封装函数内显式 setFont |
| 底部文字用大字体 | 超出 135 像素屏幕高度被裁剪 | 底部用 Font0，y 不超过 130 |
| WiFi 持续连接不设 modem sleep | 整机发烫、电池快速耗尽 | `WiFi.setSleep(true)` 开启 modem sleep；loop 里用 `vTaskDelay` 替代 `delay` 让 CPU 进 idle |
| loop 里用 delay() 忙转 | CPU 不降频，持续发热 | 用 `vTaskDelay(pdMS_TO_TICKS(10))` 替代 `delay(10)` |

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
