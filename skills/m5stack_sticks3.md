# M5StickS3 开发技能

> 面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖环境搭建、固件编译上传、IR 红外收发、按钮系统、电源管理以及已踩过的坑。

## 元数据

- **类型**: BestPractice / API Guide
- **适用场景**: 在 M5StickS3 上开发 Arduino 或 ESP-IDF 固件，尤其是 IR、按钮、电源、ES8311 音频、RMT 和 NVS
- **硬件**: M5StickS3 (ESP32-S3-PICO-1-N8R8, 8MB Flash, 8MB PSRAM)
- **创建日期**: 2026-07-29
- **最后验证**: 2026-07-29（ESP-IDF 5.5.5、`esp_codec_dev` 1.6.2）

## 这个技能解决什么问题

M5StickS3 是 M5Stack Stick 系列的最新产品，但它和前代 StickC/Plus/Plus2 在硬件上有大量不兼容之处。社区固件（Bruce、NEMO）对 StickS3 的支持不成熟，有多个已知 bug。本技能记录在 StickS3 上从零开发固件所需的全部硬约束和踩坑经验，让后续 agent 不需要重复试错。

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

## 开发环境搭建

### Arduino CLI（推荐）

```bash
# 1. 添加 M5Stack board manager URL
arduino-cli config add board_manager.additional_urls \
  https://static-cdn.m5stack.com/resource/arduino/package_m5stack_index.json

# 2. 更新索引
arduino-cli core update-index

# 3. 安装 M5Stack board package（不是 espressif 原生 esp32 core）
arduino-cli core install m5stack:esp32

# 4. 安装库
arduino-cli lib install M5Unified
arduino-cli lib install M5GFX
```

### 关键：M5Stack board package ≠ espressif esp32 core

StickS3 的 board 定义只在 M5Stack 自己的 board package 里。espressif 官方的 `esp32:esp32` core **没有** StickS3 board。两个 package 是独立的：
- `esp32:esp32` — Espressif 官方，有 ESP32-S3 DevKit 等通用板
- `m5stack:esp32` — M5Stack 定制，有 StickS3、Cardputer 等 M5 板

这条只针对 **Arduino CLI 的板型选择**。StickS3 同样支持 ESP-IDF 和
PlatformIO；它们没有 M5Stack 专用 FQBN，分别使用 target `esp32s3` 或通用
`esp32-s3-devkitc-1` 配置。无论使用哪种框架，8MB PSRAM 都是 **Octal/OPI**，
不能误配成 Quad/QSPI。Arduino 的 `m5stack_sticks3` board 默认已选好这项；
PlatformIO/ESP-IDF 项目必须显式核对。

### FQBN

```
m5stack:esp32:m5stack_sticks3
```

注意是 `m5stack_sticks3`（带前缀），不是 `m5sticks3`。

### 库版本

- M5Unified >= 0.2.12（推荐最新）
- M5GFX >= 0.2.18（推荐最新）

### Sketch 目录结构

Arduino 要求 `.ino` 文件放在同名目录下。例如：

```
src/
  my_project/
    my_project.ino   ← 文件名必须等于父目录名
```

如果直接放 `src/my_project.ino`，arduino-cli 会报 `main file missing from sketch: src/src.ino`。

## 编译与上传

```bash
# 编译
arduino-cli compile \
  --fqbn m5stack:esp32:m5stack_sticks3 \
  --build-path build/ \
  src/my_project/my_project.ino

# 上传
arduino-cli upload \
  --fqbn m5stack:esp32:m5stack_sticks3 \
  --port /dev/cu.usbmodem101 \
  --build-path build/ \
  src/my_project/my_project.ino
```

### 端口

macOS 上设备通常显示为 `/dev/cu.usbmodem*` 和 `/dev/tty.usbmodem*`；数字后缀由系统动态分配，不能写死为 `101`。官方 board 默认 USB VID:PID 为 `303A:1001`、manufacturer 为 `M5Stack`、product 为 `StickS3`；已刷的其他固件可能改写 USB Product Name。

## 按钮系统（核心踩坑区）

### 三个物理按钮

| 按钮 | GPIO / 来源 | 物理位置 | 固件可控制？ | API |
|------|-------------|----------|-------------|-----|
| BtnA | GPIO 11 | 正面 | ✅ 完全控制 | `M5.BtnA.wasPressed()` |
| BtnB | GPIO 12 | 侧面 | ✅ 完全控制 | `M5.BtnB.wasPressed()` |
| PWR/reset | M5PM1 PMIC | 侧面 | ❌ 不用于 UI | 短按复位、双击关机、长按进入下载模式 |

### BtnA / BtnB 的完整事件 API

M5Unified 的 Button_Class 提供以下检测方法（均需先调 `M5.update()`）：

```cpp
M5.BtnA.wasPressed()          // 短按（按下瞬间）
M5.BtnA.wasReleased()         // 释放
M5.BtnA.wasClicked()          // 完成一次完整点击（按下+释放）
M5.BtnA.wasHold()             // 长按（默认约 500ms+）
M5.BtnA.wasDoubleClicked()    // 双击
M5.BtnA.pressedFor(ms)        // 持续按下超过 ms 毫秒
M5.BtnA.releasedFor(ms)       // 持续释放超过 ms 毫秒
M5.BtnA.wasReleasedAfterHold() // 长按后释放
M5.BtnA.isPressed()           // 当前是否按下
```

BtnB 的 API 完全相同，把 `BtnA` 换成 `BtnB`。

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

## 红外 IR 收发（RMT 驱动）

### 硬件

- IR_TX: GPIO 46，内置 IR LED
- IR_RX: GPIO 42，内置 IR 接收器
- 两者都在设备顶部（USB-C 口对面那一端）
- 载波频率 38kHz

### 必须关闭功放

IR 接收前**必须**关闭扬声器功放，否则功放信号干扰 IR RX：

```cpp
// 方法 1（推荐）：在 M5.begin 之前禁用
auto cfg = M5.config();
cfg.internal_spk = false;
M5.begin(cfg);

// 方法 2：在 M5.begin 之后关闭
M5.Speaker.end();

// M5Unified 默认关闭 EXT_5V，内置 IR TX/RX 因此也没有供电。
// 无外部 5V 输入时必须重新开启。
M5.Power.setExtOutput(true, m5::ext_none);

// 双保险：两个关闭功放步骤都写
```

仅关闭功放不足以让 IR 工作：M5Unified 默认初始化会关闭 Grove、Hat EXT_5V
以及内置 IR TX/RX 的供电。使用 IR 收发前应显式调用
`M5.Power.setExtOutput(true, m5::ext_none)`。

### RMT 外设驱动

StickS3 的 IR **不能用 GPIO 轮询**，必须用 ESP32-S3 的 RMT 外设。使用新版 RMT API（`driver/rmt_tx.h` / `driver/rmt_rx.h`），不是旧版 `IRremote` 库。

#### TX 初始化

```cpp
rmt_tx_channel_config_t tx_cfg = {
  .gpio_num = (gpio_num_t)46,
  .clk_src = RMT_CLK_SRC_DEFAULT,
  .resolution_hz = 1000000,  // 1us per tick
  .mem_block_symbols = 64,
  .trans_queue_depth = 4,
  .flags = { .invert_out = false, .with_dma = false },
};
rmt_new_tx_channel(&tx_cfg, &tx_chan);

rmt_carrier_config_t carrier_cfg = {
  .frequency_hz = 38000,
  .duty_cycle = 0.33,
  .flags = { .polarity_active_low = false },
};
rmt_apply_carrier(tx_chan, &carrier_cfg);

rmt_copy_encoder_config_t enc_cfg = {};
rmt_new_copy_encoder(&enc_cfg, &copy_encoder);

rmt_enable(tx_chan);
```

#### RX 初始化

```cpp
rmt_rx_channel_config_t rx_cfg = {
  .gpio_num = (gpio_num_t)42,
  .clk_src = RMT_CLK_SRC_DEFAULT,
  .resolution_hz = 1000000,
  .mem_block_symbols = 128,
};
rmt_new_rx_channel(&rx_cfg, &rx_chan);

rmt_rx_event_callbacks_t cbs = { .on_recv_done = rx_callback };
rmt_rx_register_event_callbacks(rx_chan, &cbs, NULL);
rmt_enable(rx_chan);
```

#### RX 回调（ISR 上下文）

```cpp
static volatile bool rx_done = false;
static volatile size_t rx_symbol_count = 0;

bool rx_callback(rmt_channel_handle_t chan, const rmt_rx_done_event_data_t *edata, void *ctx) {
  rx_symbol_count = edata->num_symbols;
  rx_done = true;
  return false;  // 没有 wake higher priority task
}
```

返回值表示"是否需要 ISR 后调度已被唤醒的高优先级 task"，不是"是否成功"。当前
回调只写 `volatile` 标志、不唤醒 task 时返回 `false`；如果回调通过
`...FromISR` queue/semaphore API 唤醒了更高优先级 task，则返回对应的 wake 标志。

#### 启动接收

```cpp
void start_rmt_receive() {
  rx_done = false;  // 清除陈旧标志
  rmt_receive_config_t recv_cfg = {
    .signal_range_min_ns = 1000,
    .signal_range_max_ns = 20000000,
  };
  rmt_receive(rx_chan, rx_symbols, sizeof(rx_symbols), &recv_cfg);
}
```

RMT RX 是 one-shot 的：每次接收完一帧后需要重新调 `rmt_receive()` 启动下一轮。

#### 停止接收

没有直接的 `rmt_stop_receive()`。用 disable + enable 重置：

```cpp
void stop_rmt_receive() {
  rmt_disable(rx_chan);
  rmt_enable(rx_chan);
  rx_done = false;
}
```

### NEC 协议

NEC 是最常见的电视遥控器协议。一次按键发送一个 32-bit 帧：

- 引导码：9ms mark + 4.5ms space
- 32 bit 数据：每 bit = 560us mark + (560us space = 0 / 1690us space = 1)，LSB first
- 结尾：560us mark
- 总时长约 67ms

32-bit 结构：
- bit 0-7: 地址
- bit 8-15: 地址反码（标准 NEC）或地址高字节（扩展 NEC）
- bit 16-23: 命令
- bit 24-31: 命令反码

### NEC 解码校验（噪声过滤）

StickS3 内置 IR 接收器对环境红外敏感。不校验的固件（如 Bruce）会把噪声帧显示出来。有效的 NEC 校验策略：

1. 引导码时间窗口：mark 8000-10000us, space 4000-5000us
2. Repeat 帧检测：mark 8000-10000us, space 2000-3000us → 丢弃
3. 每个 bit 的 mark 校验：300-800us
4. 每个 bit 的 space 窗口校验：0 = 300-1000us, 1 = 1200-2200us（不要用单一阈值 `space > 1000`）
5. 命令反码校验：`(cmd ^ cmd_inv) == 0xFF`
6. 地址反码校验：标准 NEC 要求 `(addr ^ addr_inv) == 0xFF`；扩展 NEC 不要求（16 位地址）
7. symbol 数量校验：NEC 帧约 34 个 RMT symbol，接受范围 33-36

### IR 物理位置与距离

- 收发器在设备顶部
- Copy 时：电视遥控器对准 StickS3 顶部
- Replay 时：StickS3 顶部对准电视
- 有效距离约 2-3 米，需正对，不能穿墙

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

### 电池与电源保持

StickS3 内置 250mAh 电池。**拔插 USB 不会强制重启设备**（电池持续供电）。

StickS3 不需要 MCU GPIO4 HOLD 来维持主电源。M5PM1 自身管理主电源保持，M5Unified 的 `power_hold` pin 表也不包含 StickS3。这里不要与音频轨的 M5PM1 `LDO_HOLD` 位混淆：后者仍需在纯 ESP-IDF 音频初始化中设置。如果需要软件关机，通过 M5PM1 I2C 命令实现，而不是拉低 GPIO4。

### ES8311 与麦克风供电

ES8311 和 MEMS mic 由 `3V3_L3B_AU` 供电，不依赖 EXT_5V / BOOST 5V。纯 ESP-IDF 固件需要通过 M5PM1（I2C `0x6e`）打开 LDO，并把 GPIO2 `PYG2_L3B_EN` 配为推挽输出高电平：

```c
pmic_update(0x06, BIT(4), BIT(2)); // LDO_EN=1；可同时关闭 LED_CTRL
pmic_update(0x07, 0, BIT(5));      // LDO_HOLD=1
pmic_update(0x16, BIT(2), 0);      // GPIO2 function=GPIO
pmic_update(0x10, 0, BIT(2));      // GPIO2 direction=output
pmic_update(0x13, BIT(2), 0);      // GPIO2 push-pull
pmic_update(0x11, 0, BIT(2));      // GPIO2 high，L3B on
```

ES8311 初始化优先使用 Espressif `esp_codec_dev`。实机验证配置为：`esp_codec_dev` 1.6.2、ESP32 `I2S_ROLE_MASTER`、codec `master_mode=false`、`use_mclk=true`、256×FS MCLK、ADC mode、16-bit mono left slot、24kHz 和 36dB input gain。通过 `esp_codec_dev_open()` 完成状态转换，并用 `esp_codec_dev_read()` 读取 PCM。只手抄部分寄存器即使身份 readback 正常，也可能得到严格全零 PCM。

### EXT_5V 输出

```cpp
M5.Power.setExtOutput(true, m5::ext_none);  // 开启 BOOST/EXT_5V 输出轨
```

该轨用于 Grove、Hat 和 IR，不给 ES8311 或 MEMS mic 供电。

## 显示

```cpp
M5.Display.setRotation(3);  // landscape, 240x135
M5.Display.setFont(&fonts::FreeMonoBold9pt7b);
M5.Display.setTextColor(WHITE, TFT_BLACK);
M5.Display.clear();
M5.Display.setCursor(0, 0);
M5.Display.printf("Hello");
```

### 显示注意事项

- 屏幕分辨率 135x240，旋转 3 后是 240x135（landscape）
- `drawHeader()` 之类的封装函数要**自己设置字体**，否则会继承上一个屏幕的字体
- 底部文字不要超过 y=130（135 像素高），用 `Font0` 而非 `FreeMonoBold9pt7b` 放底部提示

## 存储（NVS / Preferences）

```cpp
#include "Preferences.h"
Preferences prefs;
prefs.begin("my_namespace", false);  // false = read/write

// 写
prefs.putULong("key", value);

// 读（带默认值）
uint32_t val = prefs.getULong("key", 0);

// blob 存储结构体（推荐，原子写入）
struct MyData { uint32_t code; uint8_t addr; };
MyData data = {0x12345678, 0x10};
prefs.putBytes("slot0", &data, sizeof(MyData));

// 读 blob
MyData out;
size_t read = prefs.getBytes("slot0", &out, sizeof(MyData));

// 检查 key 是否存在
if (!prefs.isKey("slot0")) { /* 空槽 */ }

// 删除
prefs.remove("slot0");
```

### NVS 踩坑

- 用 `putBytes` / `getBytes` 存单个 blob 比 `putULong` + `putUChar` 多次写入更安全——多次写入在断电时可能只写了一半，blob 是原子操作
- `prefs.begin()` 可能失败，检查返回值
- key 名最长 15 字符（NVS 限制）
- namespace 名最长 15 字符

## Bruce 固件在 StickS3 上的已知问题

Bruce (v1.16) 支持 StickS3 但有多个 bug：

1. **黑屏/背光初始化 bug**（GitHub issue #2371）：board 初始化代码覆盖了背光 PWM 值导致屏幕全黑，绿灯闪烁。v1.16 的 prebuilt binary 可能未包含修复。
2. **IR Read 抓到噪声**：不校验 NEC 帧，把所有 RMT 接收到的信号都显示出来，包括环境噪声。每次值不同。
3. **按钮映射问题**（GitHub issue #2148）：外部 issue 报告按钮行为异常；其物理键命名与本 skill 的 BtnA/BtnB/PWR 口径不一致，引用时不要据此改写已验证映射。
4. **RAM 报错**：麦克风录音报 "Not enough RAM"，尽管 StickS3 有 8MB PSRAM。

**结论：Bruce 在 StickS3 上不适合做严肃的 IR 工具。自己写固件更可控。**

## 三星 SolarCell Smart Remote 的 IR 问题

三星高端 TV（Neo QLED 8K 等）配的 SolarCell Smart Remote（如 VG-TM2360E / TM2360E）默认走 **Bluetooth/RF**，不是红外。这类遥控器有 IR 发射能力但出厂可能不开 IR 模式，导致 IR Copier 抓不到任何信号。

### 诊断方法

用手机摄像头（数码摄像头能看到红外光）对着遥控器顶部发射窗，按任意键。如果画面里有紫色/白色闪烁 = 红外在发；完全没反应 = Bluetooth-only 模式。

### 激活 IR 模式

同时按住遥控器上的 **Return + Play/Pause** 约 10 秒，遥控器重启并清除蓝牙配对。重新配对后 IR 功能恢复。

### 快速验证

即使蓝牙配对后，**电源键和音量键通常始终走 IR**。如果只想测试 IR Copier 能不能抓到码，先按电源键或音量键。其他智能功能键（Home、方向键等）可能只走蓝牙。

## 三星电视控制：红外 vs SmartThings API

### 红外方案（不适用于高端三星电视）

三星 Neo QLED 8K 等高端电视配的 SolarCell 遥控器（VG-TM2360E）日常**通过蓝牙控制电视**，红外只是 fallback。红外方案在以下环节全部失败：

1. **Copy 失败**：SolarCell 遥控器的私有短协议（13 symbol / 21.5ms，76us mark）无法被 StickS3 IR 接收器正确解调
2. **预设码回放失败**：IRremoteESP8266 `sendSAMSUNG()` 发送标准 Samsung32 码，电视无反应（电视不接受红外电源命令）

### SmartThings API 方案（推荐）

StickS3 通过 WiFi + SmartThings API 控制 Samsung 电视，完全绕过红外：

1. **配置**：
   - 电视添加到 SmartThings app
   - 电视设置开启 "Power On with Mobile"（Settings → All Settings → Connections → Network → Expert Settings）
   - 生成 SmartThings Personal Access Token（https://account.smartthings.com/tokens，需 `r:devices:*`、`w:devices:*`、`x:devices:*`）
   - 查询 device ID：`curl -H "Authorization: Bearer TOKEN" https://api.smartthings.com/v1/devices`

2. **API 调用**：
   - 开关：`POST https://api.smartthings.com/v1/devices/{deviceId}/commands`，body `{"commands":[{"component":"main","capability":"switch","command":"on"}]}`
   - 查状态：`GET https://api.smartthings.com/v1/devices/{deviceId}/components/main/status`
   - 音量：capability=`audioVolume`，command=`volumeUp`/`volumeDown`
   - 静音：capability=`audioMute`，command=`mute`

3. **ESP32 实现**：`WiFiClientSecure` + `setInsecure()` 跳过证书验证，`HTTPClient` 发 POST

4. **已知限制**：PAT 24 小时过期（2024-12-30 后创建的），长期方案需 OAuth2 app

### 红外协议详情（供其他电视参考）

三星遥控器使用 **Samsung32** 协议，时序与标准 NEC 类似但 header 不同。Samsung32 发送 address byte 两次（不是 address + inverse），然后 command + inverse。标准 Samsung power toggle = `0xE0E040BF`（address=0x07, command=0x02）。

IRremoteESP8266 库的 `sendSAMSUNG(0xE0E040BF, 32)` 可以发送 Samsung32 码。注意 API 是全大写 `sendSAMSUNG`，不是 `sendSamsung`。

### 重要：高端三星电视走蓝牙不走红外

三星 Neo QLED 8K 等高端电视配的 SolarCell 遥控器（VG-TM2360E）日常**通过蓝牙控制电视**，红外只是 fallback。判断方法：在楼下按遥控器，楼上电视有反应 = 蓝牙控制。这种情况下直接发送 IR 码电视不会有反应。

此外 SolarCell 遥控器的私有短协议（13 symbol / 21.5ms 帧，76us mark）无法被 StickS3 的 IR 接收器正确解调——76us 的 mark 太短，标准 IR 接收器需要至少 ~200us 连续载波才能识别。这意味着**无法可靠 copy 这个遥控器的信号**，只能用预设码发送。

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
| NVS 多次 putULong/putUChar | 断电时可能只写入部分字段 | 用 putBytes 存单个 blob |
| 拔 USB 后设备不重启 | 电池供电，拔 USB 不断电 | 双击 PWR 关机后单击开机，或短按 PWR 复位 |
| 把 ES8311 供电误认成 BOOST 5V | ES8311 身份寄存器读取失败或 codec 未上电 | 打开 M5PM1 LDO_EN/LDO_HOLD，并将 GPIO2 `PYG2_L3B_EN` 输出高电平 |
| 手抄部分 ES8311 寄存器 | 身份 readback 正常但 PCM 严格全零 | 使用 `esp_codec_dev_open()` + `esp_codec_dev_set_in_gain()` 完成 open/enable/gain 状态机 |
| 误以为需要 GPIO4 HOLD | 不必要地占用 GPIO4，或误删音频 `LDO_HOLD` | 主电源不需要 GPIO4 HOLD；音频 L3B 仍需 M5PM1 `LDO_HOLD` |
| 进入下载模式用按 BOOT 插 USB | StickS3 没有 BOOT 键，操作无效 | 连 USB 后长按侧边 PWR/reset 直到绿灯闪 |
| 把绿灯状态当成应用诊断 | 闪烁时误判崩溃，或从其他灯态推断应用正常 | 只把绿灯闪烁解释为 Download Mode；其他灯态不下结论 |
| 端口运行时消失 | 固件未启用 CDC、boot loop、低功耗或动态端口变化 | 先检查 CDC-on-boot 和串口日志；无法恢复时连接 USB，长按 PWR/reset 直到绿灯闪烁 |
| BLE HID 延时小于一个 FreeRTOS tick | 文字约 17 字符后截断，NimBLE 报 `Unable to fetch protocol_mode` | 检查 `CONFIG_FREERTOS_HZ`；100Hz 时 `pdMS_TO_TICKS(5)` 为 0。40 个 msys buffer 配合返回值检查和有界重试时，10ms 已通过连续 95 字符实机测试 |
| drawHeader 不设字体 | 继承前一个屏幕的字体，显示异常 | 封装函数内显式 setFont |
| 底部文字用大字体 | 超出 135 像素屏幕高度被裁剪 | 底部用 Font0，y 不超过 130 |

## 参考资源

- M5Stack 官方文档: https://docs.m5stack.com/en/core/StickS3
- M5Stack IR NEC 例程: https://docs.m5stack.com/en/arduino/m5sticks3/ir_nec
- M5Unified GitHub: https://github.com/m5stack/M5Unified
- M5PM1 按键与电源控制: https://github.com/m5stack/M5PM1
- Bruce 固件: https://bruce.computer
- Bruce 固件 StickS3 issue: https://github.com/BruceDevices/firmware/issues/2371

## 安装方式

将本 skill 的 GitHub URL 交给 Codex / Claude Code / Cursor / OpenCode 等 AI agent，让它：

1. 读取目标 workspace 的 `AGENTS.md` 或 `CLAUDE.md`
2. 跟随路由文件（如 `WORKSPACE.md`）
3. 将本 skill 添加到 workspace 的 skill 发现链中
4. 如果 workspace 有 `rules/skills/INDEX.md` 或 `skills/INDEX.md`，更新索引
