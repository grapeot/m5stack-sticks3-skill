# M5StickS3 开发技能

> 面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖环境搭建、固件编译上传、IR 红外收发、按钮系统、电源管理以及已踩过的坑。

## 元数据

- **类型**: BestPractice / API Guide
- **适用场景**: 在 M5StickS3 上开发 Arduino 固件，尤其是 IR 红外、按钮交互、RMT 外设、NVS 存储
- **硬件**: M5StickS3 (ESP32-S3-PICO-1-N8R8, 8MB Flash, 8MB PSRAM)
- **创建日期**: 2026-07-29

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
| BtnA | 11 | 侧键，wasPressed/wasHold/wasDoubleClicked |
| BtnB | 12 | 侧键 |
| PWR | M5PM1 | 电源键，硬件级，短按重启 |
| BOOT | 0 | 开机时 ROM 检测，运行时普通 GPIO |
| LCD MOSI | 39 | |
| LCD SCK | 40 | |
| LCD RS | 45 | |
| LCD CS | 41 | |
| LCD RST | 21 | |
| LCD BL | 38 | 背光 PWM |
| I2C SCL | 48 | BMI270 + M5PM1 共用 |
| I2C SDA | 47 | |

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

macOS 上设备显示为 `/dev/cu.usbmodem101` 和 `/dev/tty.usbmodem101`。USB Product Name 是 `StickS3_UiFlow2_`，Vendor 是 `M5Stack` (0x303A)。

## 按钮系统（核心踩坑区）

### 三个物理按钮

| 按钮 | GPIO / 来源 | 物理位置 | 固件可控制？ | API |
|------|-------------|----------|-------------|-----|
| BtnA | GPIO 11 | 正面 | ✅ 完全控制 | `M5.BtnA.wasPressed()` |
| BtnB | GPIO 12 | 侧面 | ✅ 完全控制 | `M5.BtnB.wasPressed()` |
| PWR | M5PM1 PMIC (I2C) | 侧面 | ⚠️ 有限 | 见下文 |

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

### PWR 电源键——硬件级，不可拦截

**这是最重要的踩坑。** PWR 键不是 GPIO，是 M5PM1 PMIC 通过 I2C 上报的电源事件。在 M5Unified 中：

- `M5.BtnPWR.wasClicked()` 和 `M5.BtnPWR.wasHold()` **在某些版本下可能工作**，但
- **短按 PWR 会触发硬件复位**，固件无法拦截。这不是固件 bug，是 M5PM1 的硬件行为。
- 双击 PWR = 硬件关机。

**结论：不要用 PWR 键做 UI 操作。** 只用 BtnA 和 BtnB。如果需要选择/确认/返回，用短按和双击/长按组合。

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

// 双保险：两个都写
```

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

回调返回 `false`，不是 `true`。返回值表示"是否需要 ISR 后 task scheduling"，不是"是否成功"。

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

### 电池

StickS3 内置 250mAh 电池。**拔 USB 不会断电**（电池供电），所以拔插 USB 不会强制重启设备。

### 进入 Download Mode（刷固件模式）

1. 拔掉 USB
2. **按住 BOOT 键不松手**
3. 插上 USB
4. 松开 BOOT 键

设备停在 ROM bootloader，屏幕无显示，绿灯可能闪烁。此时 esptool / arduino-cli 可以稳定连接。

### 从 Download Mode 正常启动

esptool 刷完后的 `hard reset` 在 BOOT 接地时可能无效。解决方法：
- **拔掉 USB，不按任何键，重新插上**
- 如果因为电池供电导致拔插无效，**快速按一下 PWR 键**（短按开机）
- 或者按住 BOOT 插 USB 后立刻松开 BOOT（不保持），设备会尝试从 flash 启动

### PWR 键行为

- 短按 PWR = 硬件复位（重启），固件无法拦截
- 双击 PWR = 硬件关机
- 长按 PWR = 进入 boot mode（部分固件版本）
- **固件不应依赖 PWR 键做任何 UI 功能**

### 外部供电

```cpp
M5.Power.setExtOutput(true, m5::ext_none);  // 开启 5V 外部供电
```

IR 模块和 Grove 外设可能需要外部供电。

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
3. **按钮映射问题**（GitHub issue #2148）：顶部按钮行为异常，按下后屏幕关闭 3 秒而非导航。
4. **RAM 报错**：麦克风录音报 "Not enough RAM"，尽管 StickS3 有 8MB PSRAM。

**结论：Bruce 在 StickS3 上不适合做严肃的 IR 工具。自己写固件更可控。**

## 已知陷阱汇总

| 陷阱 | 表现 | 应对 |
|------|------|------|
| 用 espressif 官方 esp32 core | 找不到 StickS3 board 定义 | 用 `m5stack:esp32` board package |
| FQBN 写成 `m5sticks3` | 编译报 unknown board | 正确是 `m5stack_sticks3`（带前缀） |
| sketch 放在 `src/foo.ino` | `main file missing from sketch: src/src.ino` | 放在 `src/foo/foo.ino` |
| 用 PWR 键做 UI | 短按触发硬件复位，设备重启 | 只用 BtnA 和 BtnB |
| IR RX 不关功放 | 接收到噪声或无信号 | `cfg.internal_spk = false` + `M5.Speaker.end()` |
| 用旧版 IRremote 库 | 不兼容 ESP32-S3 RMT | 用 `driver/rmt_tx.h` / `driver/rmt_rx.h` |
| RMT RX callback 返回 true | 不必要的 ISR 调度开销 | 返回 false |
| NEC 解码用单一阈值 `space > 1000` | 噪声和异常 space 被接受 | 用窗口：0 = 300-1000us, 1 = 1200-2200us |
| NVS 多次 putULong/putUChar | 断电时可能只写入部分字段 | 用 putBytes 存单个 blob |
| 拔 USB 后设备不重启 | 电池供电，拔 USB 不断电 | 快速按一下 PWR 键，或拔 USB 后等几秒再插 |
| 端口频繁消失 | macOS CDC 驱动 + 固件运行时 USB 不稳定 | 按 BOOT 进 download mode 再操作 |
| drawHeader 不设字体 | 继承前一个屏幕的字体，显示异常 | 封装函数内显式 setFont |
| 底部文字用大字体 | 超出 135 像素屏幕高度被裁剪 | 底部用 Font0，y 不超过 130 |

## 参考资源

- M5Stack 官方文档: https://docs.m5stack.com/en/core/StickS3
- M5Stack IR NEC 例程: https://docs.m5stack.com/en/arduino/m5sticks3/ir_nec
- M5Unified GitHub: https://github.com/m5stack/M5Unified
- Bruce 固件: https://bruce.computer
- Bruce 固件 StickS3 issue: https://github.com/BruceDevices/firmware/issues/2371

## 安装方式

将本 skill 的 GitHub URL 交给 Codex / Claude Code / Cursor / OpenCode 等 AI agent，让它：

1. 读取目标 workspace 的 `AGENTS.md` 或 `CLAUDE.md`
2. 跟随路由文件（如 `WORKSPACE.md`）
3. 将本 skill 添加到 workspace 的 skill 发现链中
4. 如果 workspace 有 `rules/skills/INDEX.md` 或 `skills/INDEX.md`，更新索引