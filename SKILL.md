---
name: ruview-dev
description: >
  RuView ESP32-S3 WiFi CSI 感知系统全链路开发维护指南。
  涵盖：固件刷写（官方/SPIRAM自编译）、WiFi配网、NVS配置、
  Tier0/Tier2数据流验证、命令行可视化工具、Web前端使用、
  多节点部署、GitHub Actions CI、常见故障排查。
  当用户在 RuView 项目中遇到 ESP32 固件、WiFi 配网、CSI 数据流、
  前端可视化、SPIRAM 内存问题时，必须使用此 skill。
---

# RuView ESP32-S3 开发维护 Skill

> **项目**: RuView (ruvnet/RuView) — WiFi CSI 人体感知系统
> **硬件**: ESP32-S3 (QFN56), 8MB Flash, 8MB PSRAM, USB-UART
> **数据协议**: ADR-018 (CSI 二进制包, magic=0xC5110001)

---

## 1. 架构速览

```
┌─────────────┐      UDP 5005      ┌─────────────────────────────┐
│  ESP32-S3   │ ═════════════════► │  电脑                        │
│  (CSI采集)   │   ADR-018 二进制   │  ├─ Node.js 命令行工具        │
│  edge_tier  │                    │  │   (mincut-person-counter)  │
│  =0 或 2    │                    │  ├─ csi-graph-visualizer.js  │
└─────────────┘                    │  ├─ Rust sensing-server (*)  │
                                   │  │   (需要编译)                │
                                   │  └─ Web UI (ui/)             │
                                   │      (需要后端中转)           │
                                   └─────────────────────────────┘
```

- **Tier 0** (`edge_tier=0`): ESP32 发送原始 CSI 帧 → 电脑做算法处理
- **Tier 2** (`edge_tier=2`): ESP32 本地计算生命体征 → 发送特征包 (br/hr/presence)

---

## 2. 环境准备（Windows）

### 2.1 必装软件

```powershell
# Python 3.12+ (带 pip)
python --version   # >= 3.12

# esptool (固件刷写)
pip install esptool==5.2.0

# NVS 分区生成工具
pip install esp-idf-nvs-partition-gen

# Node.js (可视化脚本 + 前端)
node --version     # >= v20

# 串口读取 (可选)
pip install pyserial
```

### 2.2 识别开发板

```powershell
# 列出可用串口
[System.IO.Ports.SerialPort]::GetPortNames()

# 确认芯片型号
python -m esptool --chip auto --port COM5 --baud 460800 chip_id
```

**预期输出**:
```
Chip is ESP32-S3 (QFN56) (revision v0.2)
Features: WiFi, BLE, 8MB Flash, 8MB PSRAM
Crystal is 40MHz
```

---

## 3. 固件刷写

### 3.1 官方固件（无 SPIRAM）

从 GitHub Releases 下载 `esp32-csi-node.bin` + `bootloader.bin` + `partition-table.bin` + `ota_data_initial.bin`。

```powershell
python -m esptool --chip esp32s3 --port COM5 --baud 460800 write_flash `
  0x0       bootloader.bin `
  0x8000    partition-table.bin `
  0xf000    ota_data_initial.bin `
  0x20000   esp32-csi-node.bin
```

### 3.2 SPIRAM 固件（推荐，8MB PSRAM）

**官方 Release 不含 SPIRAM 版本**，需要自编译。最简单的方式是用 **GitHub Actions**：

1. Fork 仓库到自己的 GitHub 账号
2. 在 `firmware/esp32-csi-node/` 下创建 `sdkconfig.defaults.spiram`：

```ini
CONFIG_IDF_TARGET="esp32s3"
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_display.csv"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="8MB"
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_SPIRAM_USE_CAPS_ALLOC=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=16384
CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y
CONFIG_SPIRAM_MEMTEST=y
CONFIG_SPIRAM_ALLOW_BSS_SEG_EXTERNAL_MEMORY=y
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
CONFIG_ESP_WIFI_CSI_ENABLED=y
CONFIG_BOOTLOADER_LOG_LEVEL_WARN=y
CONFIG_LOG_DEFAULT_LEVEL_INFO=y
CONFIG_LWIP_SO_RCVBUF=y
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_ESP_WIFI_EXTRA_IRAM_OPT=y
```

3. 修改 `.github/workflows/firmware-ci.yml`，在 matrix 中加入：

```yaml
matrix:
  include:
    - variant: 8mb-spiram
      sdkconfig: sdkconfig.defaults.spiram
      artifact_app: esp32-csi-node-spiram.bin
```

4. Push 到 fork，GitHub Actions 自动编译
5. 下载 artifact，四个文件（bootloader + partition-table + ota_data + app）

### 3.3 完整刷写命令（四文件）

```powershell
$port = "COM5"
$baud = 460800

python -m esptool --chip esp32s3 --port $port --baud $baud write_flash `
  0x0       bootloader.bin `
  0x8000    partition-table.bin `
  0xf000    ota_data_initial.bin `
  0x20000   esp32-csi-node-spiram.bin
```

> ⚠️ **务必使用同一套文件**（不能混用不同编译版本的 bootloader 和 app）

---

## 4. WiFi 配网（NVS 配置）

### 4.1  provision.py 用法

```powershell
cd firmware/esp32-csi-node

python provision.py `
  --port COM5 `
  --ssid "YOUR_WIFI_SSID" `
  --password "YOUR_WIFI_PASSWORD" `
  --target-ip 192.168.x.x `
  --edge-tier 0
```

**参数说明**:
| 参数 | 说明 |
|------|------|
| `--target-ip` | **电脑 WiFi 网卡的 IP**，不是以太网 IP！ESP32 通过 UDP 发数据到这个 IP 的 5005 端口 |
| `--edge-tier` | `0`=原始CSI, `1`=预处理, `2`=生命体征 |
| `--port` | ESP32 的 USB 串口号 |

### 4.2 双网卡路由陷阱（Windows 必看）

如果电脑同时有 **WiFi + 以太网**，ESP32 可能把数据发到错误的网关：

```powershell
# 检查电脑 WiFi IP（必须用 WiFi 接口的 IP）
ipconfig | Select-String -Pattern "192\.168"
```

**解决方案**: 把 `--target-ip` 设为 WiFi 接口 IP（如 `192.168.0.106`），而非以太网 IP。

### 4.3 验证配网成功

```powershell
# 方法1: 查看串口日志（按 reset 键或重新上电）
# 应看到: "WiFi connected", "target IP: 192.168.x.x", "edge_tier: 0"

# 方法2: 检查 UDP 5005 是否收到数据
netstat -ano | findstr ":5005"
```

---

## 5. 数据流验证

### 5.1 Tier 0 — 原始 CSI 数据

ESP32 发送 ADR-018 二进制包，电脑端用命令行工具解析：

```powershell
# 人员计数器
cd scripts
node mincut-person-counter.js --port 5005

# CSI 图可视化（频谱、热力图、聚类）
node csi-graph-visualizer.js --port 5005
```

**预期输出**:
```
Persons: 1  |  Frames: 45  |  Active SC: 52/64
```

> ⚠️ `csi-graph-visualizer.js` **默认端口是 5006**，必须加 `--port 5005`

### 5.2 Tier 2 — 生命体征特征

```powershell
# 切到 Tier 2（需重新配网）
python provision.py --port COM5 --ssid ... --target-ip ... --edge-tier 2

# 监听串口输出（ESP32 直接打印 br/hr/pres）
python -m serial.tools.miniterm COM5 115200
```

**串口输出示例**:
```
br=18.5 hr=72.3 pres=YES motion=0.02
```

Tier 2 时 ESP32 发送的是特征包（magic=0xC5110002），不是原始 CSI。`mincut-person-counter.js` 和 `csi-graph-visualizer.js` **不识别**这种格式。

---

## 6. 前端可视化

### 6.1 命令行工具（无需后端，立即可用）

| 工具 | 功能 | 命令 |
|------|------|------|
| `mincut-person-counter.js` | 人数检测 | `node mincut-person-counter.js --port 5005` |
| `csi-graph-visualizer.js` | CSI 频谱/热力图/聚类 | `node csi-graph-visualizer.js --port 5005` |
| `csi-spectrogram.js` | 时频谱图 | `node csi-spectrogram.js` |
| `through-wall-detector.js` | 穿墙检测 | `node through-wall-detector.js` |

### 6.2 Web UI（需要后端中转）

原项目前端在 `ui/` 目录下，但**不直接消费 UDP**，需要后端服务器：

**方案 A：Rust sensing-server（推荐，功能最全）**
```bash
cd v2
cargo build -p wifi-densepose-sensing-server --no-default-features
cargo run -p wifi-densepose-sensing-server -- --source esp32 --ui-path ../ui --http-port 3000
# 访问 http://localhost:3000/ui/index.html
```

> Windows 编译需 Visual Studio Build Tools 或 MinGW。`--no-default-features` 避免 OpenBLAS 依赖。

**方案 B：Node.js 简易桥接器（无需 Rust）**

如果无法编译 Rust，写一个 Node.js 桥接器：
- 监听 UDP 5005 接收 CSI
- 解析人员数量
- 启动 HTTP 服务器（端口 3000）服务 `ui/` 静态文件
- 启动 WebSocket（端口 8000，路径 `/ws/pose`）
- 将简化 pose 数据推送给前端

前端期望的 WebSocket 消息格式：
```json
{
  "type": "pose_data",
  "data": {
    "pose": {
      "persons": [
        {
          "id": "person_0",
          "confidence": 0.8,
          "keypoints": [],
          "bbox": null
        }
      ]
    },
    "metadata": {
      "mock_data": false,
      "source": "csi"
    }
  },
  "timestamp": 1234567890
}
```

### 6.3 Dashboard（nvsim 模拟器前端）

`dashboard/` 是 nvsim 模拟器的前端（Vite + Lit），通过 WebSocket/WASM 连接模拟器，**不直接消费 ESP32 的 UDP 数据**。

```powershell
cd dashboard
npx vite --host
# 访问 http://localhost:5173/
```

---

## 7. 调试命令速查

### 7.1 串口读取（无额外依赖）

```powershell
# PowerShell 方式
$port = New-Object System.IO.Ports.SerialPort "COM5", 115200, None, 8, One
$port.Open()
for ($i = 0; $i -lt 30; $i++) { $port.ReadLine() }
$port.Close()
```

### 7.2 端口与进程检查

```powershell
# 检查 UDP 5005 是否被监听
netstat -ano | findstr ":5005"

# 检查 TCP 5173 (Dashboard)
netstat -ano | findstr ":5173"

# 查看占用端口的进程
tasklist /FI "PID eq <PID>"

# 强制结束所有 node 进程
taskkill /F /IM node.exe
```

### 7.3 NVS 分区读取

```powershell
# 读取 NVS 分区到文件
python -m esptool --chip esp32s3 --port COM5 read_flash 0x9000 0x6000 nvs_dump.bin

# 解析（NVS 字符串值以文本形式存储）
python -c "
with open('nvs_dump.bin','rb') as f:
    text = f.read().decode('utf-8', errors='ignore')
    for kw in ['ssid','password','target_ip','edge_tier']:
        idx = text.find(kw)
        if idx != -1:
            print(text[idx:idx+64].replace('\x00',' ').strip())
"
```

---

## 8. 常见问题排查

### 8.1 WiFi 认证失败（AP 主动踢出）

**现象**: 串口反复显示 `wifi:AP send deauth, reason=2` 或 `reason=3`

**原因**: 
- 某些路由器对 ESP32 的认证握手不兼容
- 2.4GHz/5GHz 双频路由器可能分配了错误的频段

**解决**:
- 确保 WiFi 是 **2.4GHz 纯频段**（关闭 5GHz 或设不同 SSID）
- 尝试 WPA2 而非 WPA3
- 重启路由器后重新配网

### 8.2 task_wdt / 栈溢出（内存不足）

**现象**: 
```
Task watchdog got triggered. The following tasks did not reset the watchdog in time:
 - edge_dsp (CPU 1)
```

**原因**: 内部 RAM（320KB）不足，edge_dsp + LVGL 缓冲区耗尽内存

**解决**: 刷 **SPIRAM 固件**。启用 8MB PSRAM 后：
- LVGL 缓冲区从 `2x 7360 bytes (internal DMA)` → `2x 29440 bytes (PSRAM)`
- `edge_dsp` 和 `Tmr Svc` 栈溢出彻底消失

### 8.3 前端收不到数据

| 检查项 | 命令 |
|--------|------|
| UDP 5005 是否被监听 | `netstat -ano \| findstr ":5005"` |
| 防火墙是否拦截 | `Get-NetFirewallRule -Enabled True \| findstr "5005"` |
| 目标 IP 是否正确 | 确认是 WiFi 接口 IP，非以太网 |
| ESP32 是否连上 WiFi | 看串口 `WiFi connected` |
| edge_tier 是否正确 | Tier 2 时命令行工具不识别 |

### 8.4 csi-graph-visualizer.js 无数据

**最常见原因**: 默认端口是 **5006**，不是 5005！

```powershell
# ❌ 错误
node csi-graph-visualizer.js

# ✅ 正确
node csi-graph-visualizer.js --port 5005
```

---

## 9. 多节点部署

多块 ESP32 **不需要**都插到同一台电脑：

```
┌─────────────┐      WiFi      ┌─────────────────────────────┐
│  ESP32 #1   │ ═══════════════►│   电脑 192.168.0.106        │
│  (充电宝)    │                 │   node mincut-person-counter │
├─────────────┤                 │   --port 5005               │
│  ESP32 #2   │ ═══════════════►│                             │
│  (充电宝)    │                 └─────────────────────────────┘
└─────────────┘
         ↑
    同一 WiFi 路由器
```

**首次部署流程（每块板子）**:
1. USB 连电脑 → 刷固件 → `provision.py` 配网
2. 拔掉 USB → 插充电宝放到目标位置
3. 板子自动连 WiFi → 开始发送数据

**数据区分**: 当前固件 UDP 包不含设备 ID。如需区分：
- 方案 A: 多块板子配不同 `target-ip`（需多台电脑）
- 方案 B: 改固件在包前加 MAC 地址前缀
- 方案 C: 改接收脚本用 UDP 源 IP 区分

---

## 10. GitHub Actions CI（SPIRAM 固件）

### 10.1 已有配置

Fork 仓库后，`.github/workflows/firmware-ci.yml` 已配置矩阵构建：

```yaml
matrix:
  include:
    - variant: 8mb        # 标准 8MB
      sdkconfig: sdkconfig.defaults
      artifact_app: esp32-csi-node.bin
    - variant: 4mb        # 4MB Flash
      sdkconfig: sdkconfig.defaults.4mb
      artifact_app: esp32-csi-node-4mb.bin
    - variant: 8mb-spiram # 8MB + SPIRAM
      sdkconfig: sdkconfig.defaults.spiram
      artifact_app: esp32-csi-node-spiram.bin
```

### 10.2 编译步骤

```yaml
- name: Build firmware
  run: |
    . $IDF_PATH/export.sh
    cp "${{ matrix.sdkconfig }}" sdkconfig.defaults
    idf.py set-target esp32s3
    idf.py build
```

### 10.3 下载产物

Actions 运行完成后，在 **Artifacts** 中下载 `esp32-csi-node-firmware-8mb-spiram`，解压得到四个 bin 文件。

---

## 11. 关键文件索引

| 文件 | 用途 |
|------|------|
| `firmware/esp32-csi-node/provision.py` | NVS 配网脚本 |
| `firmware/esp32-csi-node/sdkconfig.defaults.spiram` | SPIRAM 编译配置 |
| `.github/workflows/firmware-ci.yml` | GitHub Actions CI |
| `scripts/mincut-person-counter.js` | 人数检测（Tier 0） |
| `scripts/csi-graph-visualizer.js` | CSI 可视化（Tier 0） |
| `scripts/csi-spectrogram.js` | 时频谱图 |
| `examples/ruview_live.py` | 串口生命体征仪表盘（Tier 2） |
| `ui/index.html` | Web 主页面（需后端） |
| `ui/viz.html` | 3D 可视化页面（需后端 WS） |
| `v2/crates/wifi-densepose-sensing-server/` | Rust 后端 |

---

## 12. 快速启动清单

```powershell
# 1. 确认硬件
python -m esptool --chip auto --port COM5 chip_id

# 2. 刷固件（四文件）
python -m esptool --chip esp32s3 --port COM5 --baud 460800 write_flash `
  0x0 bootloader.bin 0x8000 partition-table.bin `
  0xf000 ota_data_initial.bin 0x20000 esp32-csi-node-spiram.bin

# 3. 配网（target-ip 必须是 WiFi 接口 IP！）
python provision.py --port COM5 --ssid "SSID" --password "PWD" `
  --target-ip 192.168.0.106 --edge-tier 0

# 4. 验证数据流
node scripts/mincut-person-counter.js --port 5005

# 5. 可视化
node scripts/csi-graph-visualizer.js --port 5005
```
