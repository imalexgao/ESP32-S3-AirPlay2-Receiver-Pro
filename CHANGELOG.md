# Changelog

本项目所有值得记录的变更。格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)，版本号遵循[语义化版本](https://semver.org/lang/zh-CN/)。

## [v1.1] - 待发布

> 通用版专属升级（KEF EGG 专版为固定配置，不包含格式选择）。

### 新增

- **输出格式选择（Windows 式音质档位）**：后台可自由选择 采样率 × 位深 组合
  - 44.1 kHz / 16 bit —— CD 音质
  - 48 kHz / 16 bit —— DVD 音质
  - 44.1 kHz / 24 bit —— Hi-Res 入门
  - 48 kHz / 24 bit —— Hi-Res 标准（录音室音质，默认）
  - 96 kHz / 24 bit —— Hi-Res 高解析（实验性，USB 带宽满负荷）
  - 96 kHz / 16 bit —— 实验性（高采样低精度，不推荐）
- 选项旁附音质说明；设备不支持所选组合时自动回退到最近支持项并提示
- 切换在下一次 AirPlay 会话开始时生效，无需重启

### 变更

- 默认保持 48 kHz / 24-bit（与 Windows 默认一致，与 v1.0 行为相同）

### 已知限制

- 44.1 kHz 在部分声卡（如 KEF EGG）上实测音量偏小且拉音量条可能导致异常关机，选项保留但标注⚠️警告
- 96 kHz / 24-bit 的 USB 等时带宽贴近设备 MPS 上限，可能丢帧，标注"实验性"待社区实测反馈

---

## [v1.0] - 2026-09-09

初始发布（通用版 + KEF EGG 专版双版本）。

### 新增

- **AirPlay 2 / AirPlay 1 双协议接收**（fork 自 wasdwasd0105/airplay-esp32-usb，协议栈源于 rbouteiller/airplay-esp32）
- **USB Host 输出**：识别任意标准 UAC 1.0/2.0 USB 声卡（自带 USB 解码的音箱），PCM 直出由音箱解码
- **音量根治**：UAC Feature Unit 音量强制写 master + L + R 三通道——修复声道型声卡（KEF EGG）"假接受真忽略"导致的音量过小问题
- **音量解耦**：手机 AirPlay 音量与设备音量独立两级增益，设备音量可作最大上限，纯衰减无爆音
- **默认 48 kHz / 24-bit 输出**（修正上游 44.1 kHz 硬覆盖在部分声卡上小声、异常关机的硬伤）
- **完全汉化、深色近黑后台**，页面名 ESP32-AirPlay2
- **「音频编码状态」卡片**：实时显示 USB 声卡设备名、UAC 版本、采样率、位深、声道、流状态、输出峰值、AirPlay 音量、设备音量
- **遥控器全功能**：音量±（设备音量，与后台联动）；播放/暂停、上一首、下一首（DACP，AirPlay v1 模式）
- **遥控布局后台可切换**：苹果标准 / KEF EGG，立即生效
- **诊断接口**：`/api/desc`（完整 USB 描述符 dump）、`/api/audio/usb`（声道级音量回读）
- **配网体验**：热点 `esp32-airplay2_setup`（无密码）+ captive portal + 热点改名/密码/开关 + 连上家 WiFi 自动关热点、失联自动重开

### 预编译固件

- `dist/` 双版本：通用版（遥控布局默认苹果标准）+ **KEF 有源音箱专版（带 USB 解码器，如 KEF EGG）**（遥控布局写死为 KEF 键位、热点名 `esp32-airplay2`）
