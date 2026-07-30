# M5StickS3 开发技能

> 面向 AI agent 的 M5StickS3 嵌入式开发指南。覆盖环境搭建、固件编译上传、IR 红外收发、按钮系统、电源管理以及已踩过的坑。

## 元数据

- **类型**: BestPractice / API Guide
- **适用场景**: 在 M5StickS3 上开发 Arduino 或 ESP-IDF 固件，尤其是 IR、按钮、电源、ES8311 音频、RMT 和 NVS
- **硬件**: M5StickS3 (ESP32-S3-PICO-1-N8R8, 8MB Flash, 8MB PSRAM)
- **创建日期**: 2026-07-29
- **最后验证**: 2026-07-29（M5StickS3 SKU K150、ESP-IDF 5.5.5、`esp_codec_dev` 1.6.2、M5Stack Arduino core 3.3.8）

## 这个技能解决什么问题

M5StickS3 和前代 StickC/Plus/Plus2 在硬件上有大量不兼容之处。本技能记录在 StickS3 上从零开发固件时已经由官方 board-support 源码确认或在实机复现的板级约束，让后续 agent 避免把相近型号经验直接套到 StickS3。

### 边界

本技能覆盖板级 bring-up：开发环境、引脚、按钮、电源、LCD、IR、ES8311 音频、NVS，以及这些外设对实时网络/BLE 应用的约束。它不提供特定电视、云服务或语音产品的完整实现，也不把单一设备上的现象推广成某个消费电子产品系列的通用结论。

文中的结论分为三类：标注“实机验证”的内容已经在 SKU K150 上验证；引用 M5GFX/M5Unified/M5PM1 的内容来自官方 board-support 源码；其余代码片段是实现起点，必须经过下方验收，不应仅凭编译通过宣称硬件可用。

### 验收标准

一次新的 StickS3 固件 bring-up 至少满足适用项后才算完成：

| 能力 | 验收方式 |
|------|----------|
| 构建与刷机 | 从干净 build 目录为 `esp32s3`/`m5stack_sticks3` 编译成功；115200 baud 刷写完成且 image hash 校验通过 |
| 启动 | 串口确认芯片为 ESP32-S3、8MB Flash、8MB Octal PSRAM；无 boot loop |
| 按钮 | BtnA/G11、BtnB/G12 各产生一次预期事件；PWR/reset 只作为恢复路径验证 |
| LCD | 黑、白、纯红、纯绿、纯蓝色块位置和颜色正确；重复刷新无随机变色或 DMA 生命周期问题 |
| 音频 | 24kHz、16-bit、mono 采集 1 秒得到 48,000 bytes，PCM 非全零且峰值随环境声音变化；codec identity readback 单独不算成功 |
| IR（若使用） | 已关闭功放、开启 EXT_5V；已知 NEC 遥控器能稳定接收并通过反码/时序校验，回放由目标设备实际响应 |
| 网络/BLE（若使用） | 状态与日志不回显 secret；BLE HID 检查 report 返回值，并完成长于 95 字符的连续实机输入测试 |

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

- M5Stack Arduino core 3.3.8（最近验证版本）
- M5Unified >= 0.2.12
- M5GFX >= 0.2.18

为可复现构建固定实际使用的 core 和 library 版本；升级到最新版本是单独的兼容性变更，需要重新跑验收表。

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

StickS3 内置 IR 接收器对环境红外敏感。不做协议校验的固件会把环境噪声当作有效帧。有效的 NEC 校验策略：

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

### ESP-IDF 的 RGB565 颜色

StickS3 官方 M5GFX board 配置使用默认 RGB element order，裸 `esp_lcd` pixel buffer 使用标准 RGB565。不要额外选择 BGR 或手动交换 framebuffer 的红蓝字段；实测这样会让纯蓝显示成红褐色。使用标准 packing，并把 panel inversion 当作另一项独立设置：

```c
#define RGB565(r, g, b) \
  (((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3))
```

颜色排错顺序应是：先用 `0x0000`/`0xFFFF` 判断 inversion，再用纯红/纯蓝判断 element order，最后才调 palette。不要同时改 inversion、RGB/BGR 和配色，否则每次肉眼结果都无法归因。

ESP32 内存中的 `uint16_t` 是 little-endian，还必须设置 `.data_endian = LCD_RGB_DATA_ENDIAN_LITTLE`。否则 ST7789 driver 默认把 RAMCTRL 配为 big-endian：深蓝 `0x0009` 的内存字节 `09 00` 会被当成 `0x0900`（草绿），蓝色 `0x00DF` 会变成 `0xDF00`（黄）。这类“蓝变黄、深蓝变绿”是 byte endian 的强特征，不是 RGB/BGR order。

`esp_lcd_panel_draw_bitmap()` 通过 SPI panel IO 提交的 color transfer 是异步的。传入栈数组后立即复用，或传入 heap buffer 后立即 `free()`，DMA 可能继续读取已变化的内存；画面形状往往仍然正确，但颜色会随机变成黄、红等错误值。注册 `on_color_trans_done` callback，用 binary semaphore 等待传输完成后再复用 framebuffer。ESP-IDF 头文件也明确要求在该 callback 后 recycle color buffer。

StickS3 的 IPS ST7789P3 还需要 `esp_lcd_panel_invert_color(panel, true)`。如果 `0xFFFF` 白色显示为黑色、深色显示为浅色，这不可能由 RGB/BGR channel order 引起，而是遗漏了 panel INVON。应先修 inversion，再判断 palette、BGR 和 byte endian；否则会在完全错误的极性上反复猜颜色。

### 动态 UI 与实时音频

录音音量条可以直接从每个 PCM chunk 的 peak amplitude 计算，但显示必须是低优先级旁路，不能反过来拖慢采集或网络：

1. 先把 PCM 放进 WebSocket sender 的 ring buffer，再计算 meter 和绘制。
2. 不要每帧多次 `fillRect`、malloc/free 或逐条 SPI 清屏再填色；这既会闪，也会阻塞主循环。
3. 用持久化 RGB565 buffer 在内存中合成完整 meter，一次 `draw_bitmap`/DMA 提交。
4. 音频可按 20-40ms chunk 处理，但 LCD 限制到约 10fps；显示值与最新采样值分开保存，避免限帧时漏掉最后一次回落。
5. peak meter 使用快速 attack、较慢 release，可读性优于未平滑的瞬时振幅。

小屏 UI 的实测可读模式是：状态 banner 全宽贴边，深色背景配浅色状态文字；banner 下方正文统一保留安全 margin。状态页可以放到 BtnB，显示 Wi-Fi、BLE 和 token 是否配置。未认证状态接口、LCD、日志和配置页不得返回完整或截断 token；只显示 `configured` / `not configured`。配置输入框保持空白，只把非空新值解释为替换操作。

### 实时 WebSocket 语音路径

要做到松开录音键后快速出结果，不能在停止后才上传。推荐路径是：按下时异步建立 session/WebSocket，同时立即采集到 ring buffer；连接 ready 后由独立 sender task 持续发送 PCM；松开时只 drain 最后少量 backlog，再发 `commit`。显示音量、更新 LCD 或阻塞式 socket write 都不能放在 I2S capture 的关键路径上。

排查“感觉停止后才开始上传”时，分别记录按键到 first queued、first socket send、stop 时 tail drain、commit 到 transcript completed 四段时间。否则很容易把 TLS 建连、网络 backlog 或服务端 inference 延迟误判成没有流式上传。

## 存储（NVS / Preferences）

```cpp
#include "Preferences.h"
Preferences prefs;
prefs.begin("my_namespace", false);  // false = read/write

// 写
prefs.putULong("key", value);

// 读（带默认值）
uint32_t val = prefs.getULong("key", 0);

// 单 key blob，显式版本和固定宽度字段便于迁移
struct MyData { uint8_t version; uint8_t addr; uint16_t reserved; uint32_t code; };
MyData data = {1, 0x10, 0, 0x12345678};
prefs.putBytes("slot0", &data, sizeof(MyData));

// 读 blob
MyData out = {};
size_t read = prefs.getBytes("slot0", &out, sizeof(MyData));
if (read != sizeof(MyData) || out.version != 1) { /* reject or migrate */ }

// 检查 key 是否存在
if (!prefs.isKey("slot0")) { /* 空槽 */ }

// 删除
prefs.remove("slot0");
```

### NVS 踩坑

- 用单个 key 保存 blob 可降低多个 key 分别更新时出现部分新、部分旧状态的风险；不要依赖原始 C++ struct layout，必须包含版本、固定宽度字段，并校验读取长度
- `prefs.begin()` 可能失败，检查返回值
- key 名最长 15 字符（NVS 限制）
- namespace 名最长 15 字符

## BLE HID 应用约束

BLE HID 不是 StickS3 专属外设，但它常与本板的按钮、音频和网络链路组合。以下陷阱已经在 StickS3 实机出现：

- NimBLE host task 必须在 `esp_hidd_dev_init()` 安装 sync callback 后启动，否则首次 sync event 可能丢失，设备不会 advertising。
- iOS 配对需要与目标安全策略匹配；曾验证的 IDF 配置要求 `CONFIG_BT_NIMBLE_SM_LVL=2`。不要把固定 passkey 写成公开默认值。
- advertising duration 使用 `BLE_HS_FOREVER`；示例中的 180 秒会让设备之后不可发现。
- notification 会占用 mbuf 直到发出。扩大 msys pool、检查 `esp_hidd_dev_input_set()` 返回值，并使用有界重试；不要静默丢 report。
- 延时必须至少一个 FreeRTOS tick。100Hz tick 下 5ms 会被截断为 0，10ms pacing 已通过连续 95 字符实机测试。
- 不同 key 可以直接发送下一份状态 report；相同连续 key 必须先发空 report，字符串结束必须 final release。
- ESP-IDF 项目必须在包含 `project.cmake` 前确定 `IDF_TARGET=esp32s3`，否则 clean build 可能生成错误 target。

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
| 把绿灯状态当成应用诊断 | 闪烁时误判崩溃，或从其他灯态推断应用正常 | 只把绿灯闪烁解释为 Download Mode；其他灯态不下结论 |
| 端口运行时消失 | 固件未启用 CDC、boot loop、低功耗或动态端口变化 | 先检查 CDC-on-boot 和串口日志；无法恢复时连接 USB，长按 PWR/reset 直到绿灯闪烁 |
| BLE HID 延时小于一个 FreeRTOS tick | 文字约 17 字符后截断，NimBLE 报 `Unable to fetch protocol_mode` | 检查 `CONFIG_FREERTOS_HZ`；100Hz 时 `pdMS_TO_TICKS(5)` 为 0。40 个 msys buffer 配合返回值检查和有界重试时，10ms 已通过连续 95 字符实机测试 |
| 每个 BLE HID key 都发送 press + release | 打字速度只有必要 report 数量的一半 | 不同 key 可直接用下一份状态 report 替换，自动释放旧 key；相同连续 key 必须先发空 report，字符串结尾必须 release |
| 看到 BGR 配置后又手动交换 RGB565 红蓝位 | 纯蓝显示红褐色或紫色，palette 无法直觉调整 | StickS3 实机 framebuffer 使用标准 RGB565；先用纯色块单独验证 element order |
| 凭相近 ST7789 板型猜 panel 参数 | offset 对了但 inversion/order 错，反复调色仍不稳定 | 直接查 M5GFX `board_M5StickS3`：RGB order、`invert=true`、offset `(52,40)` |
| RGB565 framebuffer 未声明 little endian | 深蓝 `0x0009` 显示草绿，蓝色 `0x00DF` 显示黄色 | `esp_lcd_panel_dev_config_t.data_endian = LCD_RGB_DATA_ENDIAN_LITTLE` |
| `draw_bitmap` 返回后立刻复用或释放 buffer | 图形大致正确但颜色随机，深色 banner 变黄、浅色文字变红 | `on_color_trans_done` 发 semaphore；DMA 完成后才能改写栈/static buffer 或 `free()` heap buffer |
| StickS3 ST7789P3 未开启 color inversion | 白色显示黑色、深色显示浅色，怎么调 palette 都不对 | panel init 后调用 `esp_lcd_panel_invert_color(panel, true)`，再检查 BGR/endian |
| 音量条每个 audio chunk 多次清屏/填色 | meter 闪烁，按钮和录音链路变迟钝 | 音频先入队；静态 framebuffer 一次合成、一次 DMA，LCD 限制约 10fps |
| drawHeader 不设字体 | 继承前一个屏幕的字体，显示异常 | 封装函数内显式 setFont |
| 底部文字用大字体 | 超出 135 像素屏幕高度被裁剪 | 底部用 Font0，y 不超过 130 |

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
