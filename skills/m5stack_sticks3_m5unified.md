# M5StickS3 开发技能 — M5Unified / Arduino 部分

> 本文件覆盖 Arduino + M5Unified/M5GFX 框架下的 StickS3 开发。主 skill 在 `m5stack_sticks3.md`。

## 适用场景

使用 Arduino IDE / arduino-cli + M5Unified + M5GFX 库开发 StickS3 固件。

## 开发环境搭建

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

StickS3 的 board 定义只在 M5Stack 自己的 board package 里。两个 package 是独立的：
- `esp32:esp32` — Espressif 官方，有 ESP32-S3 DevKit 等通用板
- `m5stack:esp32` — M5Stack 定制，有 StickS3、Cardputer 等 M5 板

### FQBN

```
m5stack:esp32:m5stack_sticks3
```

注意是 `m5stack_sticks3`（带前缀），不是 `m5sticks3`。

### 库版本

- M5Stack Arduino core 3.3.8（最近验证版本）
- M5Unified >= 0.2.12
- M5GFX >= 0.2.18

### Sketch 目录结构

Arduino 要求 `.ino` 文件放在同名目录下。例如：

```
src/
  my_project/
    my_project.ino   ← 文件名必须等于父目录名
```

直接放 `src/my_project.ino` 会报 `main file missing from sketch: src/src.ino`。

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

macOS 上设备通常显示为 `/dev/cu.usbmodem*`；数字后缀由系统动态分配，不能写死为 `101`。

## 显示

### 基本用法

```cpp
M5.Display.setRotation(3);  // landscape, 240x135
M5.Display.setFont(&fonts::FreeMonoBold9pt7b);
M5.Display.setTextColor(WHITE, TFT_BLACK);
M5.Display.clear();
M5.Display.setCursor(0, 0);
M5.Display.printf("Hello");
```

### 竖屏

```cpp
M5.Display.setRotation(0);  // 竖屏 135x240
// width()=135, height()=240
```

### 文字居中（关键踩坑）

**不要用 `setCursor` + `printf` 做居中**——`printf` 的对齐行为在 M5GFX 里不可靠，会出现文字偏移。

用 `drawCenterString` 强制居中：

```cpp
M5.Display.setTextDatum(MC_DATUM);  // middle center
M5.Display.drawCenterString("ON", M5.Display.width() / 2, 90);
```

左对齐用 `drawString` + `TL_DATUM`：

```cpp
M5.Display.setTextDatum(TL_DATUM);  // top left
M5.Display.drawString("Power: ON", 5, 50);
```

### 显示注意事项

- `drawHeader()` 之类的封装函数要**自己设置字体**，否则会继承上一个屏幕的字体
- 底部文字不要超过 y=130（横屏 135 像素高），用 `Font0` 而非 `FreeMonoBold9pt7b`
- 用 `M5.Display.width()` / `M5.Display.height()` 动态算坐标，不要硬编码分辨率

## 按钮

M5Unified 的 Button_Class 提供（均需先调 `M5.update()`）：

```cpp
M5.BtnA.wasPressed()          // 按下瞬间
M5.BtnA.wasClicked()          // 完整点击（按下+释放）
M5.BtnA.wasSingleClicked()    // 单击（不与双击冲突）
M5.BtnA.wasDoubleClicked()    // 双击
M5.BtnA.wasHold()             // 长按
M5.BtnA.pressedFor(ms)        // 持续按下超过 ms
M5.BtnA.isPressed()           // 当前是否按下
```

### 单击与双击冲突

`wasPressed()` 在按下瞬间就触发，会吞掉 `wasDoubleClicked()`。解决方法：

```cpp
// 先检查双击，再检查单击
if (M5.BtnA.wasDoubleClicked()) {
  // 返回/取消
} else if (M5.BtnA.wasSingleClicked()) {
  // 选择/确认
}
```

## 红外 IR（RMT 驱动）

### 必须关闭功放 + 开启 EXT_5V

```cpp
auto cfg = M5.config();
cfg.internal_spk = false;
M5.begin(cfg);
M5.Speaker.end();
M5.Power.setExtOutput(true, m5::ext_none);
```

### RMT TX 初始化

```cpp
rmt_tx_channel_config_t tx_cfg = {
  .gpio_num = (gpio_num_t)46,
  .clk_src = RMT_CLK_SRC_DEFAULT,
  .resolution_hz = 1000000,
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

### RMT RX 初始化

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

### RMT RX 回调

```cpp
bool rx_callback(rmt_channel_handle_t chan, const rmt_rx_done_event_data_t *edata, void *ctx) {
  rx_symbol_count = edata->num_symbols;
  rx_done = true;
  return false;  // 只写 volatile 标志时返回 false
}
```

### 注意事项

- TX 和 RX 必须用同一个时钟源（都用 `RMT_CLK_SRC_DEFAULT`），否则报 `group clock conflict`
- `signal_range_min_ns` 最大 3187ns（RMT 硬件限制），设太大直接报错
- RMT RX 是 one-shot，每次接收完需重新调 `rmt_receive()`
- 停止接收：`rmt_disable()` + `rmt_enable()`

## 存储（NVS / Preferences）

```cpp
#include "Preferences.h"
Preferences prefs;
prefs.begin("my_namespace", false);

// 用单个 blob 存结构体（原子写入）
struct MyData { uint8_t version; uint32_t code; };
MyData data = {1, 0x12345678};
prefs.putBytes("slot0", &data, sizeof(MyData));

MyData out = {};
prefs.getBytes("slot0", &out, sizeof(MyData));
```

- key 名最长 15 字符
- 用 blob 而非多个 putULong，避免断电时部分写入