# M5StickS3 开发技能 — ESP-IDF 部分

> 本文件覆盖 ESP-IDF 框架下的 StickS3 板级开发。主 skill 在 `m5stack_sticks3.md`。

## 适用场景

使用 ESP-IDF（非 Arduino）开发 StickS3 固件，尤其是裸驱 ST7789P3 LCD、ES8311 音频、BLE HID 和实时 WebSocket 语音路径。

## ESP-IDF 项目配置

- `IDF_TARGET=esp32s3`（必须在包含 `project.cmake` 前确定，否则 clean build 可能生成错误 target）
- PSRAM：8MB **Octal/OPI**，不能配成 Quad/QSPI
- 没有 M5Stack 专用 FQBN，使用通用 `esp32-s3-devkitc-1` 配置

## ST7789P3 LCD 裸驱

### 面板参数（来自 M5GFX board-support 源码）

- Panel: ST7789（ST7789P3 变体）
- 分辨率: 135x240（物理竖屏）
- Offset: `(52, 40)`
- Inversion: `true`（必须调 `esp_lcd_panel_invert_color(panel, true)`）
- RGB element order: 默认（标准 RGB565）
- Write clock: 40MHz
- CS=41, RST=21, MOSI=39, SCLK=40, DC=45

### RGB565 颜色

StickS3 官方 M5GFX board 配置使用默认 RGB element order，裸 `esp_lcd` pixel buffer 使用标准 RGB565。不要额外选择 BGR 或手动交换 framebuffer 的红蓝字段。

```c
#define RGB565(r, g, b) \
  (((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3))
```

颜色排错顺序：先用 `0x0000`/`0xFFFF` 判断 inversion，再用纯红/纯蓝判断 element order，最后才调 palette。不要同时改 inversion、RGB/BGR 和配色。

### Byte Endian

ESP32 内存中的 `uint16_t` 是 little-endian，必须设置 `.data_endian = LCD_RGB_DATA_ENDIAN_LITTLE`。否则 ST7789 driver 默认把 RAMCTRL 配为 big-endian：深蓝 `0x0009` 的内存字节 `09 00` 会被当成 `0x0900`（草绿）。这类"蓝变黄、深蓝变绿"是 byte endian 的强特征。

### DMA 生命周期

`esp_lcd_panel_draw_bitmap()` 通过 SPI panel IO 提交的 color transfer 是异步的。传入栈数组后立即复用，或传入 heap buffer 后立即 `free()`，DMA 可能继续读取已变化的内存。画面形状往往正确但颜色随机变成黄、红等错误值。注册 `on_color_trans_done` callback，用 binary semaphore 等待传输完成后再复用 framebuffer。

### Panel Inversion

StickS3 的 IPS ST7789P3 需要 `esp_lcd_panel_invert_color(panel, true)`。如果 `0xFFFF` 白色显示为黑色、深色显示为浅色，是遗漏了 panel INVON。应先修 inversion，再判断 palette、BGR 和 byte endian。

## ES8311 音频

### 供电

ES8311 和 MEMS mic 由 `3V3_L3B_AU` 供电，不依赖 EXT_5V / BOOST 5V。纯 ESP-IDF 固件需要通过 M5PM1（I2C `0x6e`）打开 LDO：

```c
pmic_update(0x06, BIT(4), BIT(2)); // LDO_EN=1
pmic_update(0x07, 0, BIT(5));      // LDO_HOLD=1
pmic_update(0x16, BIT(2), 0);      // GPIO2 function=GPIO
pmic_update(0x10, 0, BIT(2));      // GPIO2 direction=output
pmic_update(0x13, BIT(2), 0);      // GPIO2 push-pull
pmic_update(0x11, 0, BIT(2));      // GPIO2 high，L3B on
```

### ES8311 初始化

优先使用 Espressif `esp_codec_dev`。实机验证配置：`esp_codec_dev` 1.6.2、ESP32 `I2S_ROLE_MASTER`、codec `master_mode=false`、`use_mclk=true`、256×FS MCLK、ADC mode、16-bit mono left slot、24kHz 和 36dB input gain。通过 `esp_codec_dev_open()` 完成状态转换，并用 `esp_codec_dev_read()` 读取 PCM。只手抄部分寄存器即使身份 readback 正常，也可能得到严格全零 PCM。

### 验收

24kHz、16-bit、mono 采集 1 秒得到 48,000 bytes，PCM 非全零且峰值随环境声音变化。codec identity readback 单独不算成功。

## BLE HID

- NimBLE host task 必须在 `esp_hidd_dev_init()` 安装 sync callback 后启动，否则首次 sync event 可能丢失
- iOS 配对需要 `CONFIG_BT_NIMBLE_SM_LVL=2`
- advertising duration 使用 `BLE_HS_FOREVER`；180 秒会让设备之后不可发现
- notification 会占用 mbuf，检查 `esp_hidd_dev_input_set()` 返回值
- 延时必须至少一个 FreeRTOS tick；100Hz tick 下 5ms 会被截断为 0，10ms pacing 已通过连续 95 字符实机测试
- 不同 key 可直接发下一份状态 report；相同连续 key 必须先发空 report，字符串结束必须 final release

## 动态 UI 与实时音频

### 音量条

1. 先把 PCM 放进 WebSocket sender 的 ring buffer，再计算 meter 和绘制
2. 不要每帧多次 `fillRect`、malloc/free；用持久化 RGB565 buffer 一次 `draw_bitmap`/DMA 提交
3. 音频可按 20-40ms chunk 处理，LCD 限制约 10fps
4. peak meter 使用快速 attack、较慢 release

### 实时 WebSocket 语音路径

按下时异步建立 session/WebSocket，同时立即采集到 ring buffer；连接 ready 后由独立 sender task 持续发送 PCM；松开时只 drain 最后少量 backlog，再发 `commit`。显示音量、更新 LCD 或阻塞式 socket write 都不能放在 I2S capture 的关键路径上。