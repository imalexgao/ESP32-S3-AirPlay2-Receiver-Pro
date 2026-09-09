# ESP32-AirPlay2 — AirPlay 2 接收器固件（ESP32-S3 + USB 声卡）

把一块 **ESP32-S3** 开发板变成 **AirPlay 2 接收器**：手机 / iPad / Mac 通过 AirPlay 投流，ESP32-S3 通过 **USB Host** 接口把 PCM 音频流给外接的 **USB 声卡（或自带 USB 解码的音箱）**，由音箱解码输出。

- 开机即开热点 **`esp32-airplay2_setup`**（无密码），手机连上后打开 **http://192.168.4.1** 完成配网
- 完全汉化、深色近黑的 Web 配置后台
- 手机 AirPlay 音量与设备音量**独立控制**（设备音量可作最大音量上限）
- 支持 AirPlay 2（默认）与 AirPlay 1 兼容模式（遥控切歌需要 v1）

---

## 基于的项目

| 项目 | 作用 |
|---|---|
| [rbouteiller/airplay-esp32](https://github.com/rbouteiller/airplay-esp32) | 独立实现的 AirPlay 2 协议栈（RTSP/HAP/SRP/DACP/mDNS 等） |
| [wasdwasd0105/airplay-esp32-usb](https://github.com/wasdwasd0105/airplay-esp32-usb) | USB Host 音频输出分支（本工程 fork 自它） |
| RemoteMapper-ESP32 | Web 配置页的交互思路参考 |

> **许可证**：上游为非商业用途许可（见上游 LICENSE）。个人 / DIY 使用没有问题，商用前请确认授权。

---

## 相对上游的改进

### 1. 音量修复：USB 声卡真实增益在声道上（KEF EGG 问题）
KEF EGG 的 USB 描述符里，Feature Unit 的 **master 通道只声明了 MUTE，真实音量增益在 L/R 声道（ch1/ch2）**。旧固件只往 master 写 0dB——EGG 不报错、回读也是 0dB（假装接受），但实际忽略，**真实增益一直卡在设备存储的低值 → 怎么都小声**。Windows 驱动会正确读描述符、把音量写到声道上，所以电脑端一直正常。

修复：音量写入**强制覆盖 master + L + R 三通道**（`uac_fu_set_volume_db`）。对标准 master 型设备无影响；对声道型设备真正生效；对严格设备（STALL）有"任一通道成功即成功"的容错。0dB 下数学上纯衰减，不会削波/爆音。

### 2. 手机音量与设备音量解耦
音频通路里两级**独立的数字增益**相乘：`最终 = PCM × 手机AirPlay音量 × 设备音量`。
- 网页滑块 / 遥控器 → 设备音量（-30..0 dB，持久化，可作**最大音量上限**）
- iOS 音量条 → AirPlay 音量（跟随连接）
- 两者互不覆盖：手机调音量不会再动后台音量条，网页调音量也不会影响手机滑块

### 3. 采样率修正（44.1k → 48k 默认）
上游把 USB Host 输出的采样率硬覆盖为 44.1 kHz。实测 KEF EGG 在 44.1 kHz 下**声音变小、且拉音量条可能导致音箱异常关机**；48 kHz / 96 kHz 正常。已移除该硬覆盖，`OUTPUT_SAMPLE_RATE_HZ` 默认 **48000**。

### 4. 后台界面
- 完全汉化 + 深色近黑主题
- 页面名 **ESP32-AirPlay2**；新增 **「音频编码状态」** 卡片（设备名、UAC 版本、采样率、位深、声道、流状态、输出电平峰值、音量）
- 新增**「遥控布局」**设置（苹果标准 / KEF EGG，立即生效）

### 5. 遥控器（HID）按键
- 修复音量键映射（EGG 的 byte[1] 布局与苹果不同）
- 新增 byte[2] 边沿检测，播放/暂停/上一首/下一首可上报
- 音量键 = 控制设备音量（±2 dB），与网页滑块联动
- 传输键走 DACP，**仅 AirPlay v1 模式可用**（AirPlay 2 用 MRP 协议，本工程未实现）

### 6. 诊断接口
- `GET /api/desc`：完整 USB 配置描述符 dump（十六进制 + 解析后的音频控制实体），排查声卡增益/控制问题用
- `GET /api/audio/usb`：附加声道级音量回读（`fu_ch1_db` / `fu_ch2_db`）、采样率回读等字段

---

## 硬件要求

- **ESP32-S3 开发板**（N8R8 / N16R8 等均可；需原生 USB-OTG 外设，ESP32 经典款不支持）
- **USB 声卡 / 自带 USB 解码的音箱**（电脑能识别为 USB 声卡即可）
- 要求声卡支持：USB Audio Class 1.0 或 2.0、PCM 16/24-bit 立体声、44.1k 或 48k（绝大多数满足）

> 如果只是调试（不接声卡），固件也能正常启动：AirPlay 会话保持，插入声卡后自动开始输出。

### 接线要点
1. **USB 声卡插在"原生 USB"口**（ESP32-S3 芯片直出的 USB-OTG，GPIO19/GPIO20，DevKitC 上通常标注 "USB"）。
2. **烧录 / 串口日志必须走另一个 USB 口**（如 UART 口 / CH343）：固件把原生 USB 口用作 USB Host 后，板载 USB-Serial-JTAG 不再可用，走外部 UART（115200）。
3. **VBUS 供电**：USB Host 模式下 VBUS 需要 5V。多数 DevKitC 把原生 USB 口的 VBUS 接到板上 5V 电源轨，总线供电的声卡可直接工作；否则请外接 5V，或在 `menuconfig → Audio Output → USB host VBUS enable GPIO` 配置 VBUS 开关。

---

## 构建与烧录

环境：PlatformIO（ESP-IDF 框架）。

```bash
# 构建（环境：esp32s3-usbhost）
pio run

# 烧录固件 + SPIFFS 网页（用 UART 口连接开发板）
pio run -t upload
pio run -t uploadfs

# 串口日志（115200）
pio device monitor
```

构建产物：
- `.pio\build\esp32s3-usbhost\firmware.bin` —— 应用固件（OTA 升级用这个）
- `.pio\build\esp32s3-usbhost\spiffs.bin` —— 网页资源

> 网页 OTA 只上传 `firmware.bin`；首次烧录或网页内容更新后，需用 `pio run -t uploadfs` 把 SPIFFS 一起写入。

### 预编译固件（`dist/` 目录）

仓库附带两个预编译版本，开箱即用（ESP32-S3 16MB Flash / 8MB PSRAM，分区见 `components/boards/partitions.csv`）：

| 版本 | 说明 | 适用 |
|---|---|---|
| **通用版** | 遥控布局可在后台切换（默认苹果标准），热点名 `esp32-airplay2_setup` | 大多数用户（推荐） |
| **KEF EGG 专版** | 遥控布局写死为 KEF EGG 键位，热点名 `esp32-airplay2`，即"音量/切歌开箱即用"的最终调试版 | 使用 KEF EGG 遥控器、不想进后台配置的用户 |

- `ESP32-AirPlay2-通用版-firmware.bin` / `-spiffs.bin`
- `ESP32-AirPlay2-KEF-EGG专版-firmware.bin` / `-spiffs.bin`
- `bootloader.bin` / `partitions.bin`（首次烧录用）

首次烧录（全量）：
```bash
esptool.py --chip esp32s3 -p COM3 -b 460800 write_flash \
  0x0 bootloader.bin 0x8000 partitions.bin \
  0x20000 ESP32-AirPlay2-通用版-firmware.bin \
  0x620000 ESP32-AirPlay2-通用版-spiffs.bin
```
已有固件只升级应用：仅烧 `0x20000` 段的 `firmware.bin`（或直接网页 OTA）。

---

## 首次使用（配网）

1. 开发板上电，等待约 5 秒出现热点 **`esp32-airplay2_setup`**（无密码）。
2. 手机连接该热点 → 自动弹出配置页（captive portal）；没弹出就手动访问 **http://192.168.4.1**。
3. 在 **WiFi 网络** 卡片点"扫描网络"，选中家里 WiFi，输入密码，点"连接 WiFi"。
4. 保存后自动重启并连接家里 WiFi（此时热点自动关闭；WiFi 连不上时会重新开热点，方便再次配网）。
5. 手机连回家里 WiFi，打开控制中心 AirPlay 图标，选择 **ESP32-AirPlay2**。

> 首次连接 iOS 设备可能要求输入 AirPlay 配对码：配对码打印在串口日志 / 日志页面中。

---

## 兼容边界与已知特例

### ① KEF EGG：真实音量在声道上
EGG 的 master 通道只有 MUTE，音量增益在 ch1(L)/ch2(R)。固件已做三通道写入，**开箱即用**；如果遇到"怎么都小声"的声卡，可用 `GET /api/desc` 查看描述符里 FU 的 `bmaControls` 分布。

### ② 遥控布局：苹果标准 vs KEF EGG（需在后台选择）
| 布局 | byte[1] 位图 | byte[2] |
|---|---|---|
| 苹果标准（默认） | 播放/暂停=0x01，音量+=0x02，音量-=0x04 | 不使用 |
| KEF EGG | 音量+=0x01，音量-=0x02 | 播放/暂停=0x20，下一首=0x40，上一首=0x80 |

两种布局在 byte[1] 上**共用比特位但含义相反**，无法自动区分。**默认苹果标准（上游行为）**；使用 KEF EGG 遥控器请在后台「设备设置 → 遥控布局」切换为 KEF EGG。

### ③ AirPlay v1 / v2
- 传输键（播放/暂停、上一首、下一首）走 DACP，**只有 AirPlay v1 模式可用**（AirPlay 2 下 iOS 改用 MRP，本工程未实现）
- 音量键两种模式都可用（本地设备音量）
- **切换模式后手机可能连不上 / 连上无声**：手机端缓存了旧服务（`_airplay` v2 / `_raop` v1），多试几次或手机端断开重连、忘掉设备即可恢复

### ④ 采样率
- 默认 48 kHz 输出（源 44.1 kHz 自动重采样）
- 部分声卡在 44.1 kHz 下异常（EGG 实测小声 + 拉音量条可能关机），如遇此类问题请保持 48 kHz

### ⑤ 电源
- USB 声卡由开发板 VBUS 供电时，务必确认 VBUS 有 5V（见"接线要点"）

---

## 目录结构（与上游一致）

```
main/               AirPlay 2 协议栈 + 应用逻辑（rtsp/ hap/ plist/ dacp/ audio/ network/）
  audio/audio_output_usb_host.c    USB Host 音频输出后端（UAC1/2 等时流 + 音量/遥控）
  network/wifi.c                   AP+STA、热点配网、断线自动恢复
  network/web_server.c             Web API（含 /api/audio/usb、/api/desc、/api/remote/layout）
data/www/           SPIFFS 网页（index.html）
components/         板级支持
config/             sdkconfig 分层配置
```

---

## 常见问题

- **没声音？** 先看后台「音频编码状态」是否显示已连接；再看日志中 `audio_uac_host` 的 Streaming 信息。若显示 `No stereo PCM iso OUT alt-setting found`，说明声卡不是标准 UAC 输出设备。
- **声音小？** 确认固件为三通道音量写入版本；用 `GET /api/audio/usb` 看 `fu_ch1_db/fu_ch2_db` 是否为 0 dB。若为负值说明声卡增益在声道上。
- **遥控键位错乱？** 后台「遥控布局」切换苹果标准 / KEF EGG。
- **切歌 / 播放暂停无效？** 切到 AirPlay v1 模式。
- **热点连不上？** 确认烧录的是 `esp32s3-usbhost` 环境；连上家里 WiFi 后热点自动关闭属正常设计。
- **AirPlay 列表里看不到设备？** 确认手机与开发板同一 WiFi，设备名 / mDNS 正常（默认 `ESP32-AirPlay2`）。
