# M5StickS3 向 iPhone 输入中文

> 调研日期：2026-07-30。目标是开发者本人在指定 iPhone 上使用：StickS3 到 iPhone 的文本交付只走 BLE，Custom Keyboard 不联网、不开启 Full Access，不要求 App Store 级通用性。ASR 在 Stick、iPhone 或既有后端完成是独立问题。

## 结论

标准 BLE/USB HID 键盘可靠传输的是键位，不是 Unicode 字符串。USB HID 虽然定义了 `Unicode Page 0x10`，但 Apple 没有公开声明 iOS/iPadOS 会把该 page 的 report 写入文本框。这个 page 仍采用两字节 UCS-2，也不能表示全部现代 Unicode。因此不要把 direct HID Unicode 放进产品主链路。

让 StickS3 先输入 literal `\u4F60\u597D`，再由 iOS 第三方键盘替换成“你好”，**值得作为第一优先级真机实验**。键盘扩展不需要拦截 HID；它只需在保持 visible/active 时，轮询 `documentContextBeforeInput`，事后看到完整 suffix，再调用 `deleteBackward()` 和 `insertText("你好")`。Apple 没有保证这个观察窗口，但本人目标机可用即可，约一小时的 spike 就能裁决。

本地路线按验证成本排序：

```text
P0: StickS3 BLE HID -> ASCII frame -> active keyboard 轮询并替换

P1: StickS3 custom GATT -> active keyboard 直连 BLE -> insertText
                            （目标机 runtime gate 通过才采用）

P2: StickS3 custom GATT -> containing app -> App Group
                                            -> active keyboard 只读并插入
```

P0 如果在目标 iPhone 和常用宿主里稳定，可以直接作为个人版最终方案。P1 更干净，因为传输最终 UTF-8、不让 ASCII frame 短暂落入正文，但旧真机报告曾显示 keyboard extension 中 CoreBluetooth 返回 `.unsupported`，所以只做最小 runtime probe。P2 代码最多，却完全避开键盘扩展直接使用 Bluetooth 的未知行为，也是前两条失败后的本地备选。

## 为什么 HID Unicode 不能当作 iOS 能力

普通 Keyboard/Keypad Page `0x07` report 描述物理键位和 modifier。iOS 再用当前布局或输入法解释它。同一份 report 在英文、德文和中文输入源下可以产生不同结果。[USB-IF HID Usage Tables](https://usb.org/sites/default/files/hut1_21.pdf) 也明确说明，创建一个 usage 不代表任何 HID host vendor 必须支持它。

Unicode Page `0x10` 的规范文字来自旧的 Unicode 1.1 / ISO 10646-1 UCS-2 模型，usage 只有 16 位。即使某个 iOS 版本实测能输入常用汉字，它仍不能直接覆盖 `U+10000` 以上字符、许多扩展汉字和 emoji，也不能据此承诺任意 Unicode。

BLE HOGP 只把同一份 HID report descriptor 放进 GATT Report Map。它改变运输方式，不会让 iOS 自动增加对 Unicode Page 的处理。

iOS 没有公开与 macOS `Unicode Hex Input` 等价的输入源。QMK 等键盘固件所谓 Unicode 输入，通常是在主机预先启用特定输入法后模拟一串普通快捷键；它不是通用 HID 字符协议，也不能把 macOS/Linux/Windows 的序列直接套到 iPhone。

## 系统拼音只能作为降级模式

StickS3 可以发送拼音字母，让当前 iOS 中文输入法产生候选。但“中文 -> 无声调拼音 -> iOS 候选”不是一一映射。人名、专有名词、生僻字和同音词会丢失信息，候选顺序还受用户历史和系统词典影响。

因此可以保留一个 HID fallback：只输入 ASCII、数字或常用短词的拼音，每个短语后停下来让用户确认候选。它不能作为任意转写结果逐字一致的路径。

## `\uXXXX` 替换方案的准确边界

Apple 公开支持 active custom keyboard 通过 `UITextDocumentProxy` 插入或删除文本：[Handling text interactions in custom keyboards](https://developer.apple.com/documentation/uikit/handling-text-interactions-in-custom-keyboards)。所以在上下文完整且光标未移动时，解析 literal 后插入中文本身可行。

断点发生在 literal 如何到达 extension：

1. 物理键盘事件进入前台宿主 App 的 responder chain；Apple 的公开 API 模型没有暴露第三方 keyboard extension 的全局 HID event tap。[Handling key presses made on a physical keyboard](https://developer.apple.com/documentation/uikit/handling-key-presses-made-on-a-physical-keyboard)
2. 外接键盘会使 onscreen keyboard 通常隐藏，不能假设 extension 仍加载或继续收到通知。
3. `documentContextBeforeInput` 只是插入点附近的可选上下文，不是完整文档或无遗漏事件流。
4. 多次 `deleteBackward()` 再 `insertText()` 不是宿主文本控件中的原子事务。回调延迟、光标移动、autocorrect 或 Web editor 都可能留下 literal 或误删正文。

第一轮不要先做自动删除。键盘扩展保持可见，用 50 Hz timer 只显示 `documentContextBeforeInput` 的末尾；Stick 发送一个固定 frame。若连续看到完整 suffix，再加入严格替换，并逐步降到 20 Hz 或 callback + polling watchdog。

P0 可以先复用当前 `\uXXXX` 输出验证机制。通过后，最终 frame 改用 Base64URL 承载 UTF-8，才能无损覆盖中文、emoji、换行和组合字符：

```text
~M5H1.<nonce>.<utf8Length>.<base64urlPayload>.<hmac64>~
```

只有闭合 sentinel、长度、Base64URL、UTF-8 和 HMAC 全部通过，且同一完整 snapshot 连续稳定两次时才删除。任何 partial、错校验、selection 非空或 document ID 变化都原样保留。正常失败应留下可见 frame，不能猜长度后误删正文。

外接 HID 会使 onscreen keyboard 通常隐藏，但 iPhone 支持用外接键盘的 Eject 显示 onscreen keyboard，并用 Control-Space 切换输入源：[Switch between keyboards with Magic Keyboard and iPhone](https://support.apple.com/en-ca/guide/iphone/iph5948b3f2e/ios)。M5 可另发 Consumer Control `AL Keyboard Layout (0x01AE)` 尝试切回虚拟键盘；generic HID 是否稳定支持仍由目标机实测。

## 两条纯 BLE 路线如何避免 Full Access

### Active keyboard extension 直接连接 GATT

这里的 BLE 是 CoreBluetooth custom GATT，不是 IP 网络。Extension 只在它可见时扫描固定 service UUID、连接 Stick、订阅 notification，收到最终 UTF-8 后直接 `insertText`。它不访问网络或 App Group，因此从 API 声明上不需要 Full Access。

当前 SDK 允许 extension target 编译 `CBCentralManager`，也没有把 API 标成 extension unavailable；但 2016 和 2021 年的真机报告都曾在 Custom Keyboard 中得到 `.unsupported`：[报告一](https://stackoverflow.com/questions/38546815/ios-corebluetooth-state-unsupported-when-using-in-ios-custom-keyboard)、[报告二](https://stackoverflow.com/questions/67467303/ios-custom-keyboard-extension-bluetooth-access)。所以先让 containing app 完成 Bluetooth 授权和普通 GATT baseline，再让 extension 只显示 `CBManager.authorization` 与 manager state：

- `.poweredOn`：继续 scan、connect、subscribe 和插入，直连路线成立。
- app 为 `.poweredOn`，extension 连续三次冷启动均为 `.unsupported`：在这台目标机停止直连路线。
- `.unauthorized`：先区分权限是否映射到 extension，不把它误判成 sandbox 禁止。

这条路线不需要 `bluetooth-central` 后台模式或 AccessorySetupKit，因为只在 keyboard active session 工作。Stick 在这个模式下不能同时注册 HID，否则 iOS 可能收起 onscreen keyboard。

### Containing app 接收 GATT，keyboard 只读 App Group

这条文本交付链路没有 IP 网络：普通 containing app 用 CoreBluetooth 接收 Stick 的最终 UTF-8，只请求 Bluetooth 权限；收到完整消息后，以 atomic rename 写入 App Group。Custom Keyboard 保持 Full Access 关闭，只读 shared container，再调用 `insertText`。若现有 Stick 固件仍通过 Wi-Fi/后端做 ASR，不影响这条 BLE 文本交付边界。

Apple 当前文档明确区分了这个方向：默认 keyboard sandbox 禁止网络并阻止**写入** containing app 的 shared group container，但**允许读取**：[Configuring open access for a custom keyboard](https://developer.apple.com/documentation/uikit/configuring-open-access-for-a-custom-keyboard)。所以数据流必须保持单向：

```text
containing app --write--> App Group --read-only--> keyboard extension
```

Keyboard 不能在共享目录写 ACK、cursor 或删除消息；它把最近处理过的 message ID 写进自己的私有 sandbox。Containing app 也不能知道宿主文本是否真正保存了 `insertText` 结果。工程上分成两层：app 完成 atomic file 后给 M5 `persisted` ACK；extension 只记录 `attempted` receipt。两者之间不存在 exactly-once 事务。

P0 只要求 containing app 前台或仍驻留后台。若以后要求 iOS 26 在 app 被系统清出内存后仍由 Bluetooth 事件重新拉起，需要 AccessorySetupKit 完成配件设置，再接 CoreBluetooth state restoration；force quit 仍不会恢复。[TN3115: Bluetooth state restoration app relaunch rules](https://developer.apple.com/documentation/technotes/tn3115-bluetooth-state-restoration-app-relaunch-rules)

不要把 `GATT -> 后台 app 写 UIPasteboard -> HID Cmd+V` 作为主路。Apple 有剪贴板写入 API，但没有公开保证 BLE 后台唤醒窗口中的写入和前台粘贴形成可靠事务；它还会覆盖用户剪贴板。

## 建议的实现顺序

1. 建一个最小 containing app + Custom Keyboard Extension，`RequestsOpenAccess=false`。先用键盘按钮验证 `insertText("你好")`。
2. 复用现有 BLE HID，做 50 Hz 只观察探针。目标机能看到完整 frame 后，再加固定 frame 替换；自建 `UITextView` 和 Notes 各连续 30 次。
3. 同一个最小工程增加 CoreBluetooth state label。先由 containing app 完成 GATT baseline，再测试 extension 是否达到 `.poweredOn`。只有 runtime gate 通过才写直连协议。
4. 如果 HID polling 不稳定且 extension CoreBluetooth 为 `.unsupported`，再做 containing app GATT -> atomic App Group file -> keyboard read-only 的前台闭环。
5. 最后才加 background central、AccessorySetupKit、本地 ASR、长消息分片和自动插入。
6. Page `0x10` 继续作为独立 protocol probe，不与上述产品路径耦合。

## Page `0x10` 最小实验

把普通键盘和 Unicode collection 放在不同 Report ID。先用 Page `0x07` 输入 `abc123` 建立 BLE/USB 基线，再分别发送 `U+0041`、`U+00E9`、`U+4F60`、`U+597D`。记录 descriptor、原始 report、发送返回值和目标文本框实际 scalar sequence；BLE 与 USB 分开测试。

只有 Apple 出现正式支持声明，或目标 iOS 范围、主要宿主 App 和重连场景都稳定通过，才重新评估 direct HID。即使通过，也只能承诺实测 BMP 子集，不能称为任意 Unicode。

## 决策摘要

| 路线 | 精确中文 | 主要问题 | 定位 |
|---|---|---|---|
| 标准 HID + 系统拼音 | 否 | 候选有歧义、状态依赖 | 降级模式 |
| HID Unicode Page `0x10` | 未知且不完整 | iOS 未公开支持、仅 16 位 | 隔离实验 |
| HID ASCII frame + extension 替换 | 条件性 | onscreen 共存、proxy 观察和非原子替换 | **第一优先级 spike** |
| active extension 直连 custom GATT | 是 | 无 Full Access runtime 可能 `.unsupported` | 低成本 runtime gate |
| custom GATT + app + App Group + keyboard | 是 | 代码与后台状态更多 | 纯本地主链备选 |
