# M5StickS3 开发技能 — 自主刷写与测试闭环

> 本文件覆盖框架无关的自主刷写闭环与测试结果契约（Arduino 与 ESP-IDF 两条路径都读这份）；
> 主 skill 在 `m5stack_sticks3.md`。

## 适用范围

做 StickS3 固件 bring-up 或持续迭代的 agent，无论使用 ESP-IDF 还是 Arduino/M5Unified 框架，均适用本文件。物理按键是恢复手段，不是正常迭代步骤。第一次 bring-up 或固件已经破坏 USB 通路时，可以请用户做一次物理恢复；设备回到正常运行态后，应立即验证并保护自主闭环。Agent 应把“自主迭代能力”本身当作 bring-up 验收项：第一次人工恢复到可运行状态后，再执行一次无按键 flash，并通过网络 health/status 或等价机器可读信号确认新固件恢复运行。不要每次刷写前都让用户长按 PWR/reset 进入下载模式。

## 自主刷写闭环

StickS3 的原生 USB Serial/JTAG 支持从正常运行态自动进入 ROM download mode。只要应用没有关闭或重配 USB、设备没有进入会让 USB 失效的睡眠状态、主机仍能看到 CDC 端口，就应直接从运行态执行自动复位、刷写和重启。

ESP-IDF 路径执行：

```bash
idf.py -p /dev/cu.usbmodemXXX -b 115200 flash
```

Arduino 路径执行：

```bash
arduino-cli upload
```

在 M5StickS3 SKU K150 + ESP-IDF 5.5.5 上，运行态到运行态的全自动循环已经实机验证。115200 baud 是已验证的可靠刷写速率；更高 baud 即使写入看似完成，也可能在 MD5/hash 校验阶段失败。不要用 `esptool run` 默认行为替代标准 flash-to-run 验收：该命令执行用户代码后仍可能按默认 `--after=hard_reset` 再次复位，观察到的最终状态容易误导。

实机验证的稳定闭环是：

```text
正常运行态
  → esptool 通过 USB Serial/JTAG 自动进入 download mode
  → 115200 baud 写入并校验 image hash
  → --after=hard_reset
  → 新固件正常运行
  → 通过网络 status/health 接口确认版本或测试结果
```

能力边界：应用若重配 GPIO19/GPIO20、关闭 USB Serial/JTAG、进入 deep sleep，或 light sleep 使 USB controller/PHY 不可响应，主机端口可能消失，此时无法从主机发起自动 download。先尝试网络控制面恢复；两条控制面都不可用时，才请求用户长按 PWR/reset 至绿灯闪烁做一次恢复刷写。不要把“已停在手工触发的 ROM download mode 后 hard reset 行为异常”和“正常运行态不能自动刷写”混为一谈。

## 刷后验收

刷写后的验收要避免改变被测状态，不要用重新打开 serial monitor 作为唯一验收：

- USB CDC/monitor 的打开、关闭和 DTR/RTS 操作可能再次触发 `USB_UART_CHIP_RESET`。
- 设备本来停在 ROM download mode 时，打开 monitor 只会再次看到 `DOWNLOAD(USB/UART0)`，不能据此证明刚才的应用从未启动。
- 优先用网络 health/status、LED 测试模式或其他独立信号确认运行。需要串口日志时，再明确把“观察应用”和“触发 reset”分开设计。

端口存在 ≠ 应用在跑：

- ROM download mode 同样枚举出 CDC 端口，与正常运行态无法从 `/dev` 区分，端口只说明芯片活着。
- 必须以应用层信号为准（READY 行、网络 health/status、屏幕行为）。
- 绿灯闪烁只解释为 Download Mode，其他灯态不下应用层结论。

download mode 刷写后设备留在 ROM 的症状与恢复：

- 症状描述：esptool 输出 "Hard resetting via RTS pin" 但设备仍停在 ROM（端口存在、`board list` 显示 "ESP32 Family Device"、串口完全静默）。
- 验收要求：刷后必须用独立信号确认应用真的启动了（READY 行/网络/行为）。
- 卡住时非物理恢复命令：

```bash
esptool -p <port> --chip esp32s3 --before default_reset --after hard_reset chip_id
```

或短按一次 PWR。

## 刷后睡眠窗口

电池产品即使最终必须进入 deep sleep，也应在刷写或其他非 deep-sleep reset 后保留约 60 秒的不可缩短 awake 窗口，让 macOS 完成 USB Serial/JTAG 枚举并给 agent 留出验证、重刷时间。用 reset reason 区分这类启动和 GPIO deep-sleep wake：前者设置全局 `sleep_not_before`，所有较短 idle timer 都不能越过它；后者走正常产品时序。只在 UI 上延迟 60 秒不够，因为期间的按钮往返或任务完成可能重新设置更早的休眠 deadline。

## 何时才请求用户介入

先检查端口、当前网络状态和 esptool 连接结果，再决定是否需要物理操作：

| 当前状态 | Agent 行为 |
|----------|------------|
| 应用正常运行，USB CDC 端口存在 | 直接自动刷写；不要请求按键 |
| 刷写成功且网络 health/status 恢复 | 自主闭环成立，继续迭代 |
| 设备已停在物理触发的 ROM download mode，刷后仍未运行 | 请求短按一次 PWR/reset；回到运行态后重新验证自动刷写 |
| CDC 端口消失，但设备仍在网络上 | 检查 light/deep sleep、USB pin/console 配置；优先通过网络命令恢复或重启 |
| CDC 和网络都不可达、固件 boot loop、USB 被重配 | 明确请求长按 PWR/reset 至绿灯闪烁，只做一次恢复刷写 |
| 电源状态不明或短按无效 | 请求双击关机再单击开机，随后恢复自主闭环 |

请求用户帮助时，要说明为什么机器已经越过自主能力边界、需要做哪个动作、预期把设备送到什么状态。用户完成后 agent 应立即继续，不把后续机器可完成的步骤再交还给用户。

## 测试结果契约

刷写成功只证明 image 可以写入，不能证明目标功能成立。高效的 agent 开发循环应形成下面的闭环：

```text
修改源码 → clean build → flash/hash verify → 读取新 build identity
        → 机器触发一个测试 → 设备执行真实业务路径
        → 返回结构化结果和原始测量 → 主机断言 → 保留失败证据或继续迭代
```

为持续实验保留一个可编译开关控制的 test mode。它应复用 production component 和 task，只增加命令解析、输入注入及结构化 telemetry；不要复制一套简化业务逻辑。USB serial 适合 bring-up，已有网络栈时也可以提供只在开发构建启用的本地 endpoint。

固件暴露的最小测试控制面必须满足以下结果契约：

- 启动后输出唯一的 `build_id` 或 git SHA，主机据此拒绝旧固件、旧端口和缓存响应。`READY` 只能在测试命令所需组件初始化完成后发送，重启后必须重新发送。
- 每条命令带 `request_id`；每次测试只有一个可关联的终态 `pass`、`fail` 或 `error`，并有明确 timeout。
- 结果至少包含 `build_id`、`request_id`、测试名、状态、耗时、关键原始测量和错误细节。不要只输出自然语言 `OK`。失败返回原始 `esp_err_t`、底层状态和关键 telemetry。
- 测试入口调用产品固件正在使用的 preprocessing、driver、inference、storage 或 network 路径；只在边界注入可控输入，不另写一套永远成功的测试实现。设备回传 raw output 与最终判断，主机用独立 expected result 断言，不要只让设备回传它自己计算的 `pass`。
- 固定输入记录长度/hash，随机流程记录 seed。主机先验证输入完整性，再判断设备输出。
- parser 对未知命令、超长输入和重复 `request_id` 给出确定错误，不因调试输入破坏正常任务。
- 性能验收同时记录时间、free internal heap、largest free block；使用 PSRAM 时另记 free PSRAM。仅记录总 heap 容易漏掉连续内存耗尽。
- 日志明确标记 `simulator`、`host` 或 `device`。只有结果确实由 StickS3 执行时才能标记 `device`。
- 未认证控制面不接收或回显 secret；测试结果中的地址、token 和用户数据必须脱敏。

协议示例（line-delimited text）：

```text
READY build_id=<git-sha-or-content-hash> target=esp32s3 reset_reason=<reason>
RUN request_id=<id> test=<name> input_len=<n> input_hash=<hash>
RESULT build_id=<id> request_id=<id> test=<name> status=<pass|fail|error> elapsed_ms=<n> ...
```

## host runner 契约

主机 runner 需满足以下契约：

- 发现动态端口（`/dev/cu.usbmodem*`），等待匹配的 `READY`，拒绝不匹配的旧 `build_id`、旧端口和缓存响应。
- 发送测试命令，验证 `request_id` 以及输入 payload 的长度与 hash。
- 解析结构化结果并进行断言；将 reset、错误结果、parse failure 和 timeout 映射为非零 exit code 表示断言失败。
- 完整原始串口记录应作为失败证据保留；CLI 不要把 timeout、设备重启或底层错误压缩成笼统的 `test failed`。

交互测试（需要人参与的手动操作，如按键、说话等）：

- host 端串口日志必须先开起来并确认在写，再请用户操作。
- 串口只在 port 打开期间保留数据，日志窗口与人的操作错位会导致关键数据段（如通话中的收发统计）永久丢失，只能重新刷写重来。

串口监听与刷写同窗口进行时的陷阱与规则：

- 同窗口进行会导致 `arduino-cli upload` 失败（esptool 连接错误 exit 2），pyserial 报 "multiple access on port"，开机日志丢失。
- 必须先刷写、后开监听；或把诊断块放 `loop()` 每几秒重跑，不依赖开机时序。

## 最小测量固件与二分

当完整应用无法解释故障时，优先做最小 measurement firmware，而不是继续猜。它只保留当前被测链路及其供电、时钟和输入，输出可量化的原始结果；确认基线后再逐项加回显示、网络、BLE、存储和睡眠。每次只改变一个主要变量，并保留最后一个通过的 firmware hash，便于二分回归和恢复。

控制面和产品功能争用 USB、GPIO19/GPIO20、内存或时序时，先用最小 measurement firmware 建立基线，再逐项加回组件。最后仍需在完整构建中重跑同一 probe，证明测量工具没有掩盖集成故障。

## 自动化边界

测试控制面可以注入按钮对应的逻辑事件、固定 PCM、网络 payload 或其他确定输入，从而减少重复人工操作，但不能把注入结果冒充物理验收。按钮逻辑可抽成同时接受 `M5.BtnA`/`M5.BtnB` 事件和 test command 的函数，便于 agent 自动遍历状态机；显示测试可以由命令切换纯色和布局。

以下边界仍需要真实设备或用户动作：

- 首次接线、USB 控制面完全失联后的恢复，以及移动、遮挡或对准设备。
- 按钮电气与机械行为：最终仍要实按 BtnA/BtnB 验证 GPIO 和 Button_Class 行为。
- 显示输出：LCD 实际颜色与布局、颜色裁切和闪烁必须在屏幕上验收。
- 音频与射频：扬声器声音、麦克风环境响应、IR/RF 目标响应。
- 功耗与电源：真实功耗测量。
- 控制面切断场景：deep sleep、USB 重配置或 GPIO19/GPIO20 用途会主动切断控制面时的最终场景验收。

进入会切断控制面的状态（如 light/deep sleep、软件重启或关闭 USB）前，应先发送终态结构化结果、`Serial.flush()` 并等待传输完成，同时保留定时唤醒、网络恢复命令或已知可工作的恢复 image。Agent 应把人工动作压缩到这些物理边界，而不是把整轮构建、刷写和日志判断交给用户。
