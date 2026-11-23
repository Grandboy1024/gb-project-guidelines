BLE API 规范关注的是移动端 APP 与智能灯硬件设备（Smart Lighting）之间在低功耗蓝牙（BLE GATT）层面的通信协议设计、广播过滤识别、数据帧封装、通用指令交互体系与状态同步机制。

BLE（Bluetooth Low Energy）采用客户端/服务端（Client/Server，即 Central/Peripheral）通信架构。APP 作为 GATT Client，智能灯设备作为 GATT Server；通过自定义 Service 与 Characteristic 实现双向数据交换，并通过 Manufacturer Specific Data 广播实现设备敏捷发现、型号判定与状态速览。

# 一、前提

为了统一移动端（iOS / Android）与嵌入式固件端的交互，提高通信可靠性、可读性、可维护性与可扩展性，所有 BLE 交互必须遵循以下基础约定：

- **传输层基础**：基于标准 BLE GATT 协议栈，依托专用 Service 与 Characteristic 进行读写与监听。
- **大端字节序**：所有多字节整型数值（包括数据长度 `Length`、校验码 `CRC16`、产品标识 `PID` 等）均严格采用大端序（Big-Endian，网络字节序，高位在前，低位在后）。
- **统一帧结构**：所有通过 GATT Characteristic 发送与接收的数据包均使用统一的“帧头 + 指令 ID + 状态码 + 长度 + 负载 + CRC16”封装。
- **可靠校验**：传输帧统一采用 CRC-16/CCITT-FALSE 校验算法，杜绝无线传输过程中的误码。
- **双向通信模型**：支持“APP 下发请求/设置并接收 ACK/响应”以及“设备状态突变主动 Notify 上报”两种模式。

## 1.1 字符集与字符串编码

- 所有文本字符串（如设备蓝牙名称 `device_name`）均统一使用 **UTF-8** 编码。
- 文本长度受通信包及固件存储限制，最大长度不得超过 32 字节（即 `N <= 32`），末尾无需包含 C 语言字符串结束符 `\0`。

## 1.2 字节序与数据类型

| 类型定义 | 字节长度 | 说明 | 示例 |
| --- | --- | --- | --- |
| `UInt8` | 1 Byte | 无符号 8 位单字节整数 | `0x01`（值为 1） |
| `Int8` | 1 Byte | 有符号 8 位整数（补码表示，-128 ~ 127） | `0xFE`（-2 dB） |
| `UInt16` | 2 Bytes | 无符号 16 位大端整数（高位在前） | `0x000A`（值为 10） |
| `UInt8[]` | N Bytes | 字节数组或特定二进制结构体 | 见各指令定义 |
| `String` | N Bytes | UTF-8 编码字节流（最大 32 字节） | `"SmartLight"` |

## 1.3 核心设计原则

1. **确定性与闭环响应**：除单向触发类动作外，APP 向设备发送的每条指令，设备均必须回复携带状态码（Status Code）的响应帧；禁止单向吞没响应。
2. **状态驱动与主动上报**：当设备发生物理操作（如物理按键开关/旋钮调光、传感器触发（PIR/光敏）、供电状态突变等）时，设备端必须通过 Notify 特征主动将最新状态同步给 APP，保证两端状态强一致。
3. **幂等性控制**：对于查询类与赋值设置类指令（如查询开关、亮度与色彩参数配置）必须设计为幂等操作；对于触发类指令（如开关灯切换、恢复出厂设置、动态效果模式切换）须在 APP 端做防连击节流保护（防抖 ≥ 500ms）。

---

# 二、BLE 广播与设备发现

设备在未被连接或处于可发现状态时，会周期性发送 BLE 广播包。APP 扫描阶段无需发起 GATT 连接即可通过厂商自定义广播数据（Manufacturer Specific Data，AD Type `0xFF`）完成设备过滤、型号判定、当前开关状态速览与设备物理 MAC 获取。

## 2.1 厂商自定义数据结构

广播数据段中 AD Type 为 `0xFF` 的厂商自定义数据（Manufacturer Specific Data）布局如下：

| 字节偏移 (Offset) | 字段名 | 长度 (Bytes) | 取值 (Hex) | 说明 |
| --- | --- | --- | --- | --- |
| Byte 0 | `Length` | 1 | `0x1E` 等 | 后面数据的总字节数（不包含 Length 本身） |
| Byte 1 | `Type` | 1 | `0xFF` | 厂商自定义数据标志（EIR / AD Type: Manufacturer Specific） |
| Byte 2~3 | `CID` | 2 | `0x22FC` 等 | 厂商识别码（Company Identifier，由 Bluetooth SIG 统一分配或企业定义，用于广播过滤识别所属硬件体系） |
| Byte 4~5 | `BID` | 2 | `0x0001` 等 | 品类/品牌识别码（Brand ID / Category ID，用于区分产品系列或产品形态） |
| Byte 6~7 | `PID` | 2 | `0x0001` ~ `0xFFFF` | 产品型号识别码（Product ID，大端序，详见 2.3 节） |
| Byte 8 | `ModelID` | 1 | `0x00` ~ `0xFF` | 商业款型与外观识别码（Model ID / Appearance & SKU ID，详见 2.1.1 节） |
| Byte 9 | `STATE` | 1 | `0x00` / `0x01` | 灯具开关与工作状态：`0x00` 关灯，`0x01` 开灯（亦支持位域扩展定义） |
| Byte 10~15 | `MAC` | 6 | 6 Bytes | 设备物理蓝牙 MAC 地址混淆数据（按字节与 `0xAD` 异或加密，详见 2.2 节） |
| Byte 16~30 | `RESERVE` | 15 | 全 `0x00` | 保留位，用于后续扩展，固件固定用 `0x00` 补位填充 |

> [!TIP]
> **字段分段内聚设计**：
> - **Byte 2 ~ Byte 8（设备身份静态元数据区）**：`CID` (厂商) $\to$ `BID` (品类) $\to$ `PID` (硬件工程平台) $\to$ `ModelID` (商业外观款型)，形成严密连续的设备身份描述体（Device Identity Descriptor）。
> - **Byte 9 ~ Byte 15（动态状态与链路信息区）**：`STATE` (灯具实时开关/工作状态) + `MAC` (设备物理蓝牙 MAC 地址)。
> - **Byte 16 ~ Byte 30（扩展保留区）**：15 字节纯净扩展空间。

### 2.1.1 商业款型/外观识别码 (ModelID) 与 PID 解耦架构

在智能照明硬件产品中，同一硬件平台（相同 PCB、LED 驱动电路、主控芯片、光学透镜结构及能力集，共享同一 `PID`）经常会衍生出多种不同的商业销售型号、物理外观形态与机身配色（CMF - Color, Material, Finish）。为了兼顾固件统一维护与 APP 端敏捷视觉呈现，必须遵循 **“功能能力绑定 PID、外观呈现绑定 ModelID”** 的解耦设计原则。

#### 1. 架构痛点与解耦价值
如果将不同外观或颜色分配为不同的 PID，会导致以下严重架构问题：
1. **固件镜像碎片化**：嵌入式团队必须为每种颜色单独编译、测试、签名并维护独立的固件二进制文件，维护成本成倍剧增。
2. **OTA 分发风险**：云端 OTA 系统需维护庞大冗余的同质固件包，极易引发跨型号错刷甚至硬件变砖。
3. **能力集与物模型膨胀**：移动端 APP 与云端物模型需为每个颜色重复配置功能能力矩阵，每次上市新颜色均需强制发版 APP。

通过在 Byte 8 引入 1 字节独立字段 `ModelID`（商业款型与外观识别码），将**物理硬件功能**与**视觉外观资产**彻底解耦：
- **`PID`（2 Bytes）定义功能能力**：决定设备支持的控制指令、调光/混色算法、动态场景模式、传感器协议及物模型属性（由 2.3 节能力集矩阵统一调度）。同一产品线的所有外观款型**共享同一 PID、同一套固件源码与 OTA 镜像**。
- **`ModelID`（1 Byte）定义视觉呈现与商业款型**：对齐 Google Fast Pair 等主流生态的 Model ID 概念，决定 APP 在**无需建立 BLE GATT 连接的广播扫描阶段**（BLE Scan）、靠近配对弹窗（Fast Pair Sheet）、设备发现列表项卡片、已连接主页中渲染对应的 3D 渲染模型、灯具体贴图、配对动画及 UI 品牌主题辅助色。

```mermaid
flowchart TD
    subgraph Adv["BLE 广播数据 (Manufacturer Data)"]
        PID["PID: 0x0101<br>(全彩 RGBCW 落地氛围灯)"]
        MID_A["ModelID: 0x00<br>(标准曜石黑款)"]
        MID_B["ModelID: 0x01<br>(极光白款)"]
    end

    subgraph Capability["业务功能与控制层 (仅依赖 PID)"]
        CapMap["能力集矩阵 (Capability Matrix)<br>• 1600万色 RGB + CCT 双白光混色<br>• 0.1%~100% 深度平滑调光<br>• 音乐律动 (拾音同步)<br>• 动态场景流转 (呼吸/彩虹/烛光)"]
        FW["统一固件与 OTA 镜像<br>(单一 Binary，全外观通用)"]
    end

    subgraph Visual["视觉与资产呈现层 (依赖 PID + ModelID)"]
        Asset_Black["曜石黑资产包<br>• 黑色灯具 3D 模型 / 渲染图<br>• 深色配对弹窗动效<br>• 专属暗调 UI 主题色"]
        Asset_White["极光白资产包<br>• 白色灯具 3D 模型 / 渲染图<br>• 浅色配对弹窗动效<br>• 专属亮调 UI 主题色"]
        Fallback["兜底机制 (Fallback)<br>未知 ModelID 优雅回退至 0x00 标准款"]
    end

    PID --> CapMap
    PID --> FW
    PID & MID_A --> Asset_Black
    PID & MID_B --> Asset_White
    MID_A -.未匹配.-> Fallback
```

#### 2. ModelID 编码规划与取值分配策略
- **数据类型**：`UInt8`（1 Byte），取值范围 `0x00` ~ `0xFF`（共支持 256 种外观形态与配色变体）。
- **默认基准约定**：
  - `0x00`：固定保留为**标准/默认基准外观（Default / Standard CMF）**。
- **分配策略选项**：
  - **策略 A：扁平枚举法（适合纯灯体配色差异产品，推荐）**：
    - `0x00`：标准款 / 曜石黑 (Classic Black，默认款)
    - `0x01`：极光白 (Aurora White)
    - `0x02`：木纹复古款 (Wood Grain Retro)
    - `0x03`：日落橙限定款 (Sunset Orange)
    - `0x80` ~ `0x8F`：特别定制 / 联名限定款（如艺术家联名款、节日限定版）
  - **策略 B：位域分层法（适合兼具外壳结构差异与机身配色的高阶产品）**：
    - **Bit 7 ~ Bit 4（高 4 位，Form Variant）**：外壳结构/造型款式（`0x0`: 标准落地款，`0x1`: 桌面台式款，`0x2`: 挂壁灯款，`0x3`: 户外地插款... 最多 16 种造型变体）。
    - **Bit 3 ~ Bit 0（低 4 位，Color CMF）**：机身配色/材质工艺（`0x0`: 曜石黑，`0x1`: 极光白，`0x2`: 钛空银，`0x3`: 香槟金... 最多 16 种配色变体）。

#### 3. 端到端工程协同与落地规范
1. **统一固件与产线烧录**：
   - 固件全系共用同一套 Binary 产物，代码内部绝不对 ModelID 进行硬编码。
   - 产线在整机组装及校准烧录环节，通过产线治具将对应 SKU 的 `ModelID`（与 `MAC`、`SN`、`PID` 等核心凭证一同）固化写入 Flash/NVRAM 专用出厂参数区（Factory Config Sector）。
   - 设备上电初始化广播数据包时，固件从 Flash 动态读取 ModelID 填充至厂商自定义数据第 8 字节（Byte 8）。
2. **APP 端解析与优雅降级（Graceful Degradation）**：
   - APP 扫描监听到广播后，同时解析 `PID` 与 `ModelID`。
   - **双键索引与安全兜底**：APP 依据 `(PID, ModelID)` 寻址对应的本地或云端动态下发的视觉资产包。若遇到用户购买了新上市配色而本地 APP 尚未发版更新对应 ModelID 资源时，APP **必须无缝降级**展示 `ModelID = 0x00` 对应的标准外观资产，**严禁发生裂图、白屏或抛出异常**。
3. **OTA 固件升级独立性**：
   - 云端 OTA 升级检查只校验 `PID` 与硬件基板版本（`hardware_version`），不与 `ModelID` 强绑定。同一 PID 下的所有外观版本均推送相同的固件版本升级包。

#### 4. 广播报文数据实例对比
以高阶落地氛围灯（CID=`0x22FC`，BID=`0x0001`，PID=`0x0101`，STATE=`0x00`，MAC=`AA:BB:CC:DD:EE:FF`）为例，不同款型在 BLE 广播帧中的 16 进制报文对比如下：

| 商业款型 | ModelID (Hex) | 厂商自定义数据完整报文 Hex (共 31 字节) | 差异说明 |
| --- | :---: | --- | --- |
| **标准曜石黑款** | `0x00` | `1E FF FC 22 01 00 01 01 00 00 07 16 61 70 43 52 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | Byte 8 为 `0x00` (默认标准黑) |
| **极光白款** | `0x01` | `1E FF FC 22 01 00 01 01 01 00 07 16 61 70 43 52 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | Byte 8 为 `0x01` (极光白) |
| **日落橙限定款** | `0x03` | `1E FF FC 22 01 00 01 01 03 00 07 16 61 70 43 52 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | Byte 8 为 `0x03` (日落橙) |

*(注：Byte 10~15 MAC 地址字段已按 2.2 节算法完成异或混淆处理，`0xAA ^ 0xAD = 0x07` 等)*

## 2.2 设备物理蓝牙 MAC 地址混淆与还原算法

在 iOS 系统中，CoreBluetooth 框架出于隐私考虑不会向 APP 暴露外设的真实物理 MAC 地址，仅提供临时随机生成的 UUID。为了保证 iOS 与 Android 端能够获取唯一确定的物理硬件标识用于设备绑定、云端鉴权、Mesh 组网编组与本地鉴权，固件在广播中显式携带 6 字节设备物理蓝牙 MAC 地址（Byte 10~15），并采用固定异或掩码（XOR Mask `0xAD`）进行简单防嗅探混淆存储。

### 混淆与还原公式：

$$\text{Original\_MAC}[i] = \text{Raw\_MAC}[i] \oplus 0\text{xAD} \quad (0 \le i \le 5)$$

```typescript
/**
 * 设备物理蓝牙 MAC 地址解析还原示例 (TypeScript)
 * @param obfuscatedBytes 从广播 Byte 10~15 提取的 6 字节 MAC 字段
 * @returns 标准格式 MAC 地址，如 "AA:BB:CC:DD:EE:FF"
 */
function decodeClassicMac(obfuscatedBytes: Uint8Array): string {
  const macHex: string[] = [];
  for (let i = 0; i < 6; i++) {
    const originalByte = obfuscatedBytes[i] ^ 0xAD;
    macHex.push(originalByte.toString(16).padStart(2, '0').toUpperCase());
  }
  return macHex.join(':');
}
```

## 2.3 产品型号识别码 (PID) 与能力集 (Capability) 映射机制

APP 端根据广播中的 `PID`（大端 2 字节）映射对应的智能灯产品型号与设备能力集（Capability Matrix），以实现照明功能支持与控制面板的动态装配，避免针对特定型号硬编码逻辑。

### 2.3.1 PID（硬件功能）与 ModelID（商业款型与外观）职责分离模型

为了应对“相同 PID 存在不同商业款型/外观配色”的业务场景，系统架构遵循严格的双键分离设计：

- **PID (Product ID)** $\to$ **映射能力集 (Capability Matrix)**：决定设备具备的照明功能（如 CCT 色温范围、RGB 混色引擎、调光阶数、音乐律动拾音、传感器支持等）。同一产品型号的所有外观款型共享相同的 PID 和能力集。
- **ModelID (Model ID)** $\to$ **映射视觉资产与商业款型 (Visual Assets & Theme)**：决定设备在扫描发现、弹窗配对及控制面板中呈现的 3D 渲染图、配色贴图与 UI 主题辅助色。
- **动态寻址与兜底流程**：APP 先按 `PID` 实例化底层设备控制器与能力集，再按 `(PID, ModelID)` 加载对应皮肤与外观模型；若 `ModelID` 匹配失败，自动回退到 `(PID, 0x00)` 标准款外观。

### 2.3.2 设备能力集解耦模型

APP 建立连接后，依据 PID 获取预置或云端动态下发的智能灯设备能力元数据，按需启用对应的指令集与控制面板：

| 产品分类示例 | 示例 PID | 硬件形态 | 支持的核心能力矩阵 (Capability Matrix) |
| --- | --- | --- | --- |
| 单色温调光灯 (基础款) | `0x0001` | 筒灯 / 射灯 / 步道地插灯 | 0%~100% 线性调光、开关定时器、延时关灯、夜灯微光模式 |
| 双色温 CCT 调光灯 (进阶款) | `0x0101` | 智能吸顶灯 / 护眼台灯 / 面板灯 | 2700K~6500K 色温连续调节、0.1%~100% 深度平滑调光、生物节律日光模拟、预设工作/阅读/休闲场景 |
| 全彩 RGBCW 氛围灯 (高阶款) | `0x0201` | 落地氛围灯 / 幻彩灯带 / RGBW 壁灯 | 1600万色 RGB + CCT 双白光独立混光、音乐律动拾音、流动/呼吸/彩虹跑马动态特效、自定义场景色彩 |
| 智能户外多功能泛光/庭院灯 (旗舰款) | `0x0301` | 太阳能泛光灯 / 庭院火炬景观灯 | PIR 人体红外感应联动、环境光敏检测 (日落自启/日出休眠)、多灯 BLE 组网群控同步、电池/太阳能电量状态监测 |

---

# 三、GATT 通信架构与服务定义

设备与 APP 通过标准的 BLE GATT 协议通信。在建立物理连接后，APP 需发现自定义服务并开启相关特征的监听。

## 3.1 服务与特征分配 (UUID)

| 类型 (Type) | 属性 (Properties) | 128-bit 标准 UUID | 16-bit 别名 | 说明 |
| --- | --- | --- | --- | --- |
| **Control Service** | - | `0000FF77-0000-1000-8000-00805F9B34FB` | `0xFF77` | 核心控制服务 |
| **Write Characteristic** | `Write` / `Write Without Response` | `0000FF88-0000-1000-8000-00805F9B34FB` | `0xFF88` | APP 向设备下发控制指令与查询请求 |
| **Notify Characteristic** | `Notify` | `0000FF99-0000-1000-8000-00805F9B34FB` | `0xFF99` | 设备向 APP 发送指令回复及主动状态上报 |

> [!IMPORTANT]
> APP 建立连接后，必须第一时间向 `0xFF99` 特征的 CCCD（Client Characteristic Configuration Descriptor，UUID: `0x2902`）写入 `0x0001` 以启用 Notification 监听；否则将无法收到任何设备响应与上报数据。

## 3.2 链路交互参数与 MTU 协商

1. **MTU 交换 (Exchange MTU)**：
   - BLE 默认 ATT_MTU 为 23 字节（有效负载 20 字节）。
   - 连接建立并开启通知后，APP 端建议立即请求 MTU 协商，推荐协商至 **247 字节**（或 512 字节），保证长指令可在单个 ATT PDU 内完整发送，避免固件端进行分包重组。
2. **连接间隔 (Connection Interval)**：
   - 推荐配置：7.5ms ~ 20ms，平衡响应延迟与功耗。

---

# 四、通用数据帧协议规范

无论是 APP 发给设备的指令，还是设备发给 APP 的回复与上报，所有通信数据均封装在统一格式的数据帧中。

## 4.1 帧结构定义

数据帧由 **帧头（Frame Head）**、**指令数据（Command Data Body）** 和 **校验码（Checksum）** 三部分组成：

```mermaid
flowchart LR
    subgraph CRC["CRC-16 校验计算范围 (共 5 + N 字节)"]
        direction LR
        subgraph Head["帧头 Frame Head (5 Bytes)"]
            direction LR
            F1["Head Flag<br><b>0xFE</b> (1 Byte)"]
            F2["Command ID<br>指令标识 (1 Byte)"]
            F3["Status Code<br>状态码 (1 Byte)"]
            F4["Data Length<br>负载长度 (2 Bytes)"]
            F1 --> F2 --> F3 --> F4
        end
        subgraph Body["数据负载 Payload (N Bytes)"]
            F5["Command Data<br>业务参数体 (N Bytes)"]
        end
        Head --> Body
    end
    subgraph Tail["校验码 Checksum (2 Bytes)"]
        F6["CRC-16<br>CCITT-FALSE (2 Bytes)"]
    end
    Body --> Tail
```

### 字段说明与布局表

| 逻辑分段 | 字段名 (Field) | 长度 (Bytes) | 取值范围 | 说明 | CRC-16 范围 |
| :--- | :--- | :---: | :--- | :--- | :---: |
| **帧头 (5 Bytes)** | `Head Flag` | 1 | `0xFE` | 帧起始标志符，固定为 `0xFE`，双向统一 | 参与计算 |
| **帧头 (5 Bytes)** | `Command ID` | 1 | `0x01` ~ `0xFE` | 控制指令标识符，依据号段分配规范定义 | 参与计算 |
| **帧头 (5 Bytes)** | `Status Code` | 1 | `0x00` ~ `0x05` | 执行状态/错误码（APP 发送置 0，设备应答时返回结果） | 参与计算 |
| **帧头 (5 Bytes)** | `Command Data Length` | 2 | `0x0000` ~ `0xFFFF` | 数据负载长度 $N$（大端序），不含帧头与校验码 | 参与计算 |
| **数据负载 (N Bytes)**| `Command Data` | N | 二进制字节流 | 业务参数体；当 Length = 0 时此字段缺省 | 参与计算 |
| **校验码 (2 Bytes)** | `CRC16` | 2 | `0x0000` ~ `0xFFFF` | 循环冗余校验码（大端序），覆盖前 $5+N$ 字节 | **校验存放位** (不计入) |

## 4.2 校验码算法规范 (CRC-16/CCITT-FALSE)

本规范采用工业通用的 **CRC-16/CCITT-FALSE** 算法：
- **多项式 (Polynomial)**：$x^{16} + x^{12} + x^5 + 1$（`0x1021`）
- **初始值 (Initial Value)**：`0xFFFF`
- **异或输出 (XorOut)**：`0x0000`
- **输入反转 (RefIn)**：False
- **输出反转 (RefOut)**：False

### 4.2.1 C 语言标准实现

```c
#include <stdint.h>
#include <stddef.h>

/**
 * CRC-16/CCITT-FALSE 计算函数
 * @param buffer 输入数据指针（从 Head Flag 0xFE 开始）
 * @param size   计算长度（5 字节帧头 + N 字节负载）
 * @return 16位 CRC 校验码
 */
uint16_t ble_crc16(const uint8_t *buffer, uint32_t size)
{
    uint16_t crc = 0xFFFF; // 初始值为 0xFFFF
    if (buffer != NULL && size > 0) {
        while (size--) {
            crc = (crc >> 8) | (crc << 8);
            crc ^= *buffer++;
            crc ^= ((uint8_t)crc) >> 4;
            crc ^= crc << 12;
            crc ^= (crc & 0xFF) << 5;
        }
    }
    return crc;
}
```

### 4.2.2 Java / Kotlin 标准实现

```java
public class BleCrc16 {
    /**
     * CRC-16/CCITT-FALSE 计算
     */
    public static int calculate(byte[] data, int size) {
        if (data == null) return 0;
        int crc = 0xFFFF;
        for (int i = 0; i < data.length && i < size; i++) {
            crc = ((crc >> 8) | (crc << 8)) & 0xFFFF;
            crc ^= (data[i] & 0xFF);
            crc ^= ((crc & 0xFF) >> 4) & 0xFFFF;
            crc ^= (crc << 12) & 0xFFFF;
            crc ^= ((crc & 0xFF) << 5) & 0xFFFF;
        }
        return crc & 0xFFFF;
    }
}
```

### 4.2.3 TypeScript / JavaScript 标准实现

```typescript
/**
 * CRC-16/CCITT-FALSE 计算
 */
export function calculateCrc16(data: Uint8Array, size?: number): number {
  const len = size !== undefined ? Math.min(size, data.length) : data.length;
  let crc = 0xFFFF;
  for (let i = 0; i < len; i++) {
    crc = ((crc >> 8) | (crc << 8)) & 0xFFFF;
    crc ^= (data[i] & 0xFF);
    crc ^= ((crc & 0xFF) >> 4) & 0xFFFF;
    crc ^= (crc << 12) & 0xFFFF;
    crc ^= ((crc & 0xFF) << 5) & 0xFFFF;
  }
  return crc & 0xFFFF;
}
```

## 4.3 状态码 / 错误码定义 (Status Code)

当设备向 APP 返回响应时，通过帧头中的 `Status Code`（第 3 字节）标识指令执行状态。

| 状态码 (Hex) | 常量定义 | 含义说明 | 典型场景 |
| --- | --- | --- | --- |
| `0x00` | `SUCCESS` | 成功 | 指令已正确接收并生效，无异常。 |
| `0x01` | `INVALID_COMMAND` | 无效指令 | 设备固件不支持该 `Command ID`。 |
| `0x02` | `LENGTH_ERROR` | 数据长度错误 | `Command Data Length` 与指令所需参数长度不符。 |
| `0x03` | `CHECKSUM_ERROR` | 校验码错误 | 数据帧传输误码，CRC-16 校验失败。 |
| `0x04` | `PARAMETER_ERROR` | 参数错误 | 参数值超出合法区间（如数值超出有效范围）。 |
| `0x05` | `CONDITION_NOT_MET` | 环境不具备，不能执行 | 设备当前处于不允许该操作的状态（如硬件处于过热保护状态禁止开启最高亮度）。 |

> [!NOTE]
> APP 发送给设备的所有指令，设备端必须回复响应报文（至少携带对应状态码），以形成确定性通信闭环。若校验失败或格式错误，设备端直接回复 `0x02` 或 `0x03` 错误帧。

---

# 五、通用控制指令设计规范与扩展指南

为确保协议具备高度的通用性、长期演进能力与多人协作一致性，后续所有新增控制指令均必须严格遵循本章制定的通用规范进行设计、分配与落地。

## 5.1 指令交互分类模型

所有在 BLE GATT 通道上传输的业务指令，划分为以下四种基础类型：

```mermaid
flowchart TD
    Class[控制指令交互分类]
    Class --> Q[1. 属性查询类 Query]
    Class --> S[2. 属性设置类 Set]
    Class --> A[3. 动作触发类 Action]
    Class --> N[4. 状态主动上报类 Notify]

    Q --> Q_Desc["APP 发起 (Len=0) -> BT 回复全量属性 (幂等)"]
    S --> S_Desc["APP 下发期望值 -> BT 执行并回显最新状态 (幂等)"]
    A --> A_Desc["APP 触发瞬态操作 -> BT 回复 ACK 后异步执行 (需防抖)"]
    N --> N_Desc["硬件物理突变 -> BT 主动推送到 Notify 特征 (事件驱动)"]
```

### 5.1.1 属性查询类 (Query / Read)
- **通信方向**：`APP -> BT`（查询请求），`BT -> APP`（属性响应）。
- **设计约定**：
  - 查询请求帧的 `Command Data Length` 通常为 `0x0000`（无数据载荷）；若支持子属性或条件过滤，可按需携带子索引。
  - 响应帧中 `Status Code` 填 `0x00`，`Command Data` 返回当前最新属性值。
  - **幂等性**：严格幂等，多次执行不改变设备内部状态。

### 5.1.2 属性设置类 (Set / Write)
- **通信方向**：`APP -> BT`（设置请求），`BT -> APP`（设置响应）。
- **设计约定**：
  - 请求帧中携带待生效的目标属性数据（单个数值或复合结构体）。
  - 设备处理成功后，回复携带 `Status Code = 0x00` 的响应帧；推荐在响应帧的 `Command Data` 中回显当前生效的最新数据（支持部分参数因硬件裁剪后的实际折衷值）。
  - **幂等性**：多次下发相同数值的设置帧，设备最终状态保持一致。

### 5.1.3 动作触发类 (Action / Trigger)
- **通信方向**：`APP -> BT`（动作请求），`BT -> APP`（执行应答）。
- **设计约定**：
  - 用于控制设备执行一次性瞬态动作（如开关切换、恢复出厂设置、软重启、动态特效单次触发等）。
  - 设备收到后立即执行参数与前置条件校验，先行回复 ACK 响应帧，随后异步调度硬件动作。
  - **防抖控制**：因无线重传或用户界面连击可能导致动作多次执行，APP 端必须建立防抖或节流机制（防抖保护时间推荐 $\ge 500\text{ms}$）。

### 5.1.4 状态主动上报类 (Event / Notify)
- **通信方向**：`BT -> APP`（主动推送）。
- **设计约定**：
  - 当设备因物理按键/旋钮操作、传感器触发（PIR/光敏）、供电状态变化或硬件中断等产生状态跃迁时，固件端通过 `0xFF99` 特征主动推送对应 `Command ID` 的数据帧。
  - 主动上报帧的 `Status Code` 固定置 `0x00`，`Command Data` 为变更后的最新全量状态，保证 APP 端 UI 与硬件设备强一致。

## 5.2 Command ID 编码与号段分配规范

Command ID 占用 1 字节（`0x01` ~ `0xFE`），为了防止功能交叉、指令碰撞以及方便多团队协同，采用按功能领域分区的号段规划：

| 号段区间 (Hex) | 业务功能领域 (Domain) | 说明与典型范例 |
| --- | --- | --- |
| `0x00` | **系统保留 (System Reserved)** | 严禁分配给任何业务指令。 |
| `0x01` ~ `0x1F` | **系统与设备基础管理** | 设备供电与电池状态、蓝牙名称配置、出厂复位、系统时钟同步、固件版本信息、OTA 升级握手等。 |
| `0x20` ~ `0x3F` | **基础照明与开关调光控制** | 灯光总开关 On/Off、主亮度无级调节 (0~100%)、平滑开闭淡入淡出时长配置、夜灯模式、倒计时延时关灯等。 |
| `0x40` ~ `0x5F` | **色彩、色温与光学引擎** | CCT 色温调节 (2700K~6500K)、RGB 全彩混色、HSV 色相饱和度调节、RGBCW 独立通道控制、色彩平滑渐变速率等。 |
| `0x60` ~ `0x7F` | **动态场景与传感器联动** | 动态灯光情景模式（呼吸/流动/闪烁/烛光/彩虹）、音乐律动拾音灵敏度与节奏算法、PIR 人体红外感应参数、光敏传感器阈值等。 |
| `0x80` ~ `0xDF` | **业务预留与群组扩展** | 多灯组网编队同步、生物节律日光日程安排、定时情景任务链等高阶功能预留号段。 |
| `0xE0` ~ `0xFE` | **产测与私有调试号段** | LED 积分球色温校准、光衰与老化测试、PWM 频率校准、传感器灵敏度标定等内部工程指令（APP 生产包严禁暴露）。 |
| `0xFF` | **协议保留字 (Reserved)** | 广播标志与特殊控制字，禁止作为业务 Command ID 分配。 |

## 5.3 载荷数据体 (Payload) 设计规范

指令的 `Command Data` 字段包含具体的控制参数。为了兼顾嵌入式系统的资源限制与移动端的解析效率，必须遵守以下设计原则：

1. **紧凑打包与内存对齐**：
   - 数据字段必须按固定字节偏移紧凑排列，**严禁引入任何编译器自动填充的 Padding 字节**。固件端结构体定义必须强制使用紧凑打包修饰符（如 GCC/Clang 下使用 `__attribute__((packed))`，MSVC 下使用 `#pragma pack(push, 1)`）。
2. **多字节数值大端序**：
   - 载荷内出现的所有多字节整型（`UInt16`、`UInt32` 等）必须强制遵循大端字节序（Big-Endian，高位在前）。
3. **位域与状态压缩 (Bitfield)**：
   - 对于多个布尔量或“状态+百分比”的复合场景，应充分利用单字节的不同比特位，降低空中传输载荷（例如：高 1 位表示充放电状态，低 7 位表示 0~100 的百分比数值）。
4. **定长优先与变长受限**：
   - 复合参数优先采用定长结构体（Fixed-Length Struct）。
   - 若必须包含变长数据（如字符串文本），必须在字段前显式声明 1 字节的长度前缀，或直接依托帧头的 `Command Data Length` 予以推导，单次文本传输最大长度不得超过 32 字节。

## 5.4 协议向前/向后兼容性设计铁律

随着硬件固件迭代与 APP 版本更新，两端经常出现版本不一致的情况。所有指令的设计与演进必须遵循三项兼容性铁律：

```
旧版本载荷: [ 字段 A (1B) ] [ 字段 B (2B) ]
新版本载荷: [ 字段 A (1B) ] [ 字段 B (2B) ] [ 字段 C (1B) ] <-- 新增参数只允许在尾部追加！
```

1. **尾部追加原则**：后续若需为已有指令增加参数，只能在原 Payload 末尾依次追加新增字节，**严禁在既有字段中间插入新字段，严禁更改既有字段的字节偏移与语义**。
2. **短包容忍度（APP 兼容旧固件）**：当 APP 接收设备上报或响应时，如果实际收到的 `Command Data Length` 小于最新协议定义的长度（说明设备端为旧版固件），APP 必须以安全默认值填充缺失字段并继续解析，**严禁抛出异常、崩溃或丢弃整帧**。
3. **长包容忍度（固件兼容新 APP）**：当固件接收 APP 设置指令时，如果实际收到的数据长度大于固件当前支持的长度（说明 APP 已升级为包含新特性的版本），旧固件仅解析应用自身已知的前置字段，忽略尾部多余字节，**严禁返回 `0x02`（LENGTH_ERROR）**。

## 5.5 指令交互闭环与异常处理规范

1. **全链路响应闭环**：
   - APP 每次向写入特征（`0xFF88`）下发数据帧后，设备必须在完成解析与执行后，通过通知特征（`0xFF99`）向 APP 回复响应帧。
   - 响应帧中的 `Command ID` 必须与请求帧严格一致。
2. **错误状态码规范回复**：
   - 当设备检测到异常情况时，必须回复错误响应帧，其格式统一为：
     - `Command ID`：保持与下发指令一致。
     - `Status Code`：置为对应的非零错误码（`0x01` ~ `0x05`）。
     - `Command Data Length`：固定置为 `0x0000`（无数据载荷）。
     - `Command Data`：缺省。
     - `CRC16`：依据前 5 字节帧头重新计算生成的校验码。

---

## 5.6 新增控制指令标准制定模板

当项目需要新增任何控制指令时，设计者可直接参考并复制下述模板结构进行协议编写，经嵌入式固件与移动端双方评审通过后归档生效：

> [!TIP]
> **编写指引**：复制下述内容至业务协议文档中，将占位符 `[xxx]` 替换为具体业务定义，并按需增删负载字段。

---

### 5.x [指令中文名称] (`0x[Command_ID]`)

[一句话说明该指令的业务功能、应用场景及触发时机]

- **Command ID**：`0xXX`（取值需符合 5.2 节号段分配规范）
- **指令类别**：[属性查询 / 属性设置 / 动作触发 / 状态主动上报]
- **支持通信方向**：
  - `APP -> BT`：[说明 APP 下发的场景与意图]
  - `BT -> APP`：[说明 BT 回复的响应或主动推送的场景]

#### 1. 数据字段结构 (Payload Definition)

**请求报文负载 (APP -> BT)**：
| 字节偏移 (Offset) | 字段名 (Field) | 类型 (Type) | 长度 (Bytes) | 取值范围 (Range) | 默认值 (Default) | 说明 (Description) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Byte 0 | `param_example_1` | `UInt8` | 1 | `0x00` ~ `0xFF` | `0x00` | 字段说明 |
| Byte 1~2 | `param_example_2` | `UInt16`| 2 | `0x0000` ~ `0xFFFF` | `0x0000` | 大端序整型说明 |

**响应 / 上报报文负载 (BT -> APP)**：
| 字节偏移 (Offset) | 字段名 (Field) | 类型 (Type) | 长度 (Bytes) | 取值范围 (Range) | 说明 (Description) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Byte 0 | `param_example_1` | `UInt8` | 1 | `0x00` ~ `0xFF` | 当前生效的状态值 |
| Byte 1~2 | `param_example_2` | `UInt16`| 2 | `0x0000` ~ `0xFFFF` | 当前生效的大端序整型值 |

*(注：若为无载荷的查询或 ACK 响应，注明“负载长度为 0，Command Data 缺省”)*

#### 2. 报文示例 (Message Examples)
1. **APP 发起查询/设置**：
   - 负载 Hex：`01 00 64`
   - 完整报文 Hex：`FE XX 00 00 03 01 00 64 <CRC16>`
2. **BT 正常响应 / 主动上报**：
   - 负载 Hex：`01 00 64`
   - 完整报文 Hex：`FE XX 00 00 03 01 00 64 <CRC16>`
3. **BT 异常应答 (如参数越界)**：
   - 完整报文 Hex：`FE XX 04 00 00 <CRC16>`

#### 3. 业务状态码与边界约束
- **参数合法性校验**：[详细列出各字段在什么条件下判定为参数错误 0x04]
- **前置执行条件**：[例如是否需要特定硬件模式支持，否则返回 0x05 CONDITION_NOT_MET]

---

## 5.7 标准参考范例 (Canonical Reference Implementations)

以下提供三类最典型智能照明业务场景的标准规范范例，供开发者参考与对照实现：

### 范例一：单属性读写与双向同步指令（以主亮度无级调节为例）

- **Command ID**：`0x21`（属于基础照明与调光控制号段）
- **指令类别**：属性查询、属性设置、状态主动上报
- **支持方向**：`APP -> BT`，`BT -> APP`

#### 字段结构定义
| 字节偏移 | 字段名 | 类型 | 长度 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Byte 0 | `target_brightness` | `UInt8` | 1 Byte | `0x00` ~ `0x64` (0~100) | 目标亮度百分比（`0x00` 关灯/最低微光，`0x64` 100% 全亮） |

#### 典型报文流向
1. **APP 查询当前主亮度**：
   - 报文：`FE 21 00 00 00 F9 06`（Data Length = 0）
2. **BT 回复当前亮度为 10% (`0x0A`)**：
   - 报文：`FE 21 00 00 01 0A 2F 91`
3. **APP 设置主亮度为 10% (`0x0A`)**：
   - 报文：`FE 21 00 00 01 0A 2F 91`
4. **BT 执行成功回复 ACK**：
   - 报文：`FE 21 00 00 01 0A 2F 91`（Status = 0x00，回显当前值）
5. **设备本地物理旋钮/触控调节到 10% 时的主动上报**：
   - 报文：`FE 21 00 00 01 0A 2F 91`

---

### 范例二：复合结构体下发与同步指令（以全彩混光多通道参数配置为例）

- **Command ID**：`0x41`（属于色彩、色温与光学号段）
- **指令类别**：复合属性设置与状态同步
- **支持方向**：`APP -> BT`，`BT -> APP`

#### 字段结构定义 (固定 5 字节负载)
| 字节偏移 | 字段名 | 类型 | 长度 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Byte 0 | `mode_select` | `UInt8` | 1 Byte | `0x00` ~ `0x03` | 照明光学模式（0: 纯白光 CCT, 1: RGB 彩光, 2: 动感流光, 3: 自定义模式） |
| Byte 1~4 | `param_array` | `Int8[4]` | 4 Bytes | `-10` ~ `+10` | 4 通道色彩增益/权重配置（R, G, B, W 各通道调光微调电平） |

#### 典型报文流向
- **APP 下发配置 (模式 1，增益分别为 2, 3, 4, 5)**：
  - 数据负载：`01 02 03 04 05`
  - 完整报文：`FE 41 00 00 05 01 02 03 04 05 96 8B`
- **BT 成功应用并回显配置**：
  - 完整报文：`FE 41 00 00 05 01 02 03 04 05 96 8B`
- **BT 参数越界错误响应 (如参数传入 20)**：
  - 完整报文：`FE 41 04 00 00 1B 8B`

---

### 范例三：单向动作触发与确认指令（以出厂复位为例）

- **Command ID**：`0x11`（属于系统管理号段）
- **指令类别**：动作触发类（单向动作，带防抖保护）
- **支持方向**：`APP -> BT`

#### 字段结构定义
- 无参数体，`Command Data Length` 固定为 `0x0000`。

#### 典型报文流向
1. **APP 发送系统出厂复位指令**：
   - 完整报文：`FE 11 00 00 00 8C E1`
2. **设备回复确认并异步执行**：
   - 完整报文：`FE 11 00 00 00 8C E1`（Status Code = 0x00）
   - **设备端行为**：设备收到指令后先通过 Notify 发出成功响应帧，清除已绑定的网络/配对数据，恢复出厂默认照明状态（如 100% 亮度 4000K 暖白光），延时 100ms 调度复位流程，主动断开 BLE 链路并重启广播。

---

# 六、通信流程与容错机制

## 6.1 连接与初始化握手时序

APP 与设备从扫描发现到建立双向受控状态，必须严格遵循以下时序流程：

```mermaid
sequenceDiagram
    autonumber
    participant App as 移动端 APP (Client)
    participant Dev as 硬件设备 (Server)

    Note over App,Dev: 1. 广播发现阶段
    Dev-->>App: BLE ADV (0xFF 厂商数据: CID, BID, PID, ModelID, STATE, MAC)
    App->>App: 校验 CID，按 PID 映射能力集，按 ModelID 匹配外观模型与视觉资产 (0x00 兜底)

    Note over App,Dev: 2. GATT 连接与通道建立
    App->>Dev: 发起 BLE 物理连接 (Connect)
    Dev-->>App: 连接建立成功 (Connected)
    App->>Dev: 发现服务 (Discover Services: 0xFF77)
    App->>Dev: 开启通知 (Write CCCD 0x2902 = 0x0001 on Char 0xFF99)
    Dev-->>App: CCCD 写入成功 (Notify Enabled)
    App->>Dev: 协商 MTU (Request Exchange MTU = 247)
    Dev-->>App: 返回协商后 MTU (e.g. 247)

    Note over App,Dev: 3. 初始全量状态同步
    App->>Dev: 查询系统通用属性 (如 Command ID 0x01)
    Dev-->>App: 回复对应状态数据帧 (Status=0x00)
    App->>Dev: 查询灯光开关、亮度、色彩与场景模式 (如 Cmd 0x20/0x40)
    Dev-->>App: 回复对应照明状态数据帧 (Status=0x00)
    App->>App: 状态装配完成，呈现可控状态
```

## 6.2 请求与响应交互模型 (Request-Response)

针对 APP 发起的设置与查询指令，设备均以携带执行状态的报文进行应答：

```mermaid
sequenceDiagram
    autonumber
    participant App as APP (Client)
    participant Dev as 硬件设备 (Server)

    App->>Dev: Write on 0xFF88: FE 21 00 00 01 0A 2F91 (设置主亮度为 10%)
    Dev->>Dev: 校验帧头(FE)、长度(1)及 CRC16(2F91)
    alt 校验正确且执行成功
        Dev-->>App: Notify on 0xFF99: FE 21 00 00 01 0A 2F91 (Status=0x00 成功)
    else 参数越界 (例如数值超出定义范围)
        Dev-->>App: Notify on 0xFF99: FE 21 04 00 00 7CC8 (Status=0x04 参数错误)
    else 校验失败
        Dev-->>App: Notify on 0xFF99: FE 21 03 00 00 F958 (Status=0x03 校验错误)
    end
```

## 6.3 事件驱动的主动上报模型 (Device-Initiated Notification)

当用户在设备本体上进行物理按键、旋钮或传感器感应触发时，设备通过 `0xFF99` 特征主动将变化后的状态推送给 APP：

```mermaid
sequenceDiagram
    autonumber
    participant User as 用户物理操作
    participant Dev as 硬件本体 (Server)
    participant App as APP (Client)

    User->>Dev: 旋转物理旋钮调节亮度至 10% (0x0A)
    Dev->>Dev: 本地硬件 PWM 调光立即生效，光照平滑过渡
    Dev-->>App: Notify on 0xFF99: FE 21 00 00 01 0A 2F91 (主动上报最新亮度)
    App->>App: 刷新 UI 界面亮度组件展示为 10%
```

## 6.4 异常处理与工程容错策略

1. **超时与重发机制**：
   - APP 发出需应答指令后，启动 **1500ms** 单次超时计时器。
   - 若 1500ms 内未收到对应 `Command ID` 的响应，允许最多重试 **2 次**。
   - 超过 3 次仍无应答，APP 判定为通信链路故障，提示用户或检测蓝牙连接健康度。
2. **防重放与粘包拆包（Frame Slicing）**：
   - 虽然通过 MTU 协商后绝大多数指令无需拆包，但为应对复杂操作系统或终端未成功协商 MTU（仍为 23 字节）的情况，APP 与固件均必须实现基于 `0xFE` 帧头与 `Command Data Length` 的流式接收状态机。
   - 只有接收完整 $5 + N + 2$ 字节并通过 CRC-16 校验后，才将整帧提交给上层业务解析器。
3. **断线与自动重连**：
   - 蓝牙物理链路断开时，APP 必须立刻清理正在等待响应的请求队列，避免重连后历史请求造成状态污染。
   - 链路重连成功并重新开启 Notify 后，APP 必须全量重新查询一次设备基础状态，重新对齐端侧数据。

---

# 七、通用测试向量与校验参考 (Test Vectors & Verification)

以下测试向量提供通用协议报文的原始 16 进制流及 CRC 校验计算结果，用于固件与 APP 研发联调、算法移植及单元测试基准比对。

## 7.1 CRC-16/CCITT-FALSE 算法标准基准

| 测试输入 | 长度 (Bytes) | 期望 CRC16 (大端 Hex) | 说明 |
| :--- | :--- | :--- | :--- |
| ASCII `"123456789"` | 9 | `29 B1` | 国际通用 CCITT-FALSE 标准校验基准 |
| `FE 01 00 00 00` | 5 | `97 46` | 无载荷标准帧头校验 |

## 7.2 通用协议帧基准测试向量表

| 序号 | 业务场景 | 方向 | 原始负载 Hex (帧头+数据) | CRC16 (大端) | 完整报文 Hex |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | **无载荷通用查询帧 (Cmd 0x01)** | APP -> BT | `FE 01 00 00 00` | `97 46` | `FE010000009746` |
| 2 | 通用成功响应 (返回 2 字节数据 00 64) | BT -> APP | `FE 01 00 00 02 00 64` | `EB 3C` | `FE010000020064EB3C` |
| 3 | **单字节亮度设置帧 (Cmd 0x21, 值为 10%)** | APP -> BT | `FE 21 00 00 01 0A` | `2F 91` | `FE210000010A2F91` |
| 4 | 单字节亮度设置成功回显 / 主动上报 | BT -> APP | `FE 21 00 00 01 0A` | `2F 91` | `FE210000010A2F91` |
| 5 | **多参数混光配置帧 (Cmd 0x41, 5 字节负载)** | APP -> BT | `FE 41 00 00 05 01 02 03 04 05` | `96 8B` | `FE410000050102030405968B` |
| 6 | **无参动作触发帧 (Cmd 0x11, 复位/动作)** | APP -> BT | `FE 11 00 00 00` | `8C E1` | `FE110000008CE1` |
| 7 | **变长文本设置帧 (Cmd 0x0A, 字符串 SmartLight)** | APP -> BT | `FE 0A 00 00 0A 53 6D 61 72 74 4C 69 67 68 74` | `03 F0` | `FE0A00000A536D6172744C6967687403F0` |
| 8 | *通用应答 - 无效指令 (INVALID_COMMAND, Status 0x01)* | BT -> APP | `FE 25 01 00 00` | `5D C9` | `FE250100005DC9` |
| 9 | *通用应答 - 长度错误 (LENGTH_ERROR, Status 0x02)* | BT -> APP | `FE 21 02 00 00` | `CE 68` | `FE21020000CE68` |
| 10 | *通用应答 - 校验错误 (CHECKSUM_ERROR, Status 0x03)* | BT -> APP | `FE 21 03 00 00` | `F9 58` | `FE21030000F958` |
| 11 | *通用应答 - 参数错误 (PARAMETER_ERROR, Status 0x04)* | BT -> APP | `FE 21 04 00 00` | `7C C8` | `FE210400007CC8` |
| 12 | *通用应答 - 环境不满足 (CONDITION_NOT_MET, Status 0x05)* | BT -> APP | `FE 21 05 00 00` | `4B F8` | `FE210500004BF8` |
