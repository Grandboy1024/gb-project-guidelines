# BLE API 通信协议规范 V1.1（正式工程版）

> **文档状态：** 工程基线草案（帧结构、广播、核心基础控制与测试向量已冻结，可用于 APP / 固件 / 产测联调；安全算法套件、部分能力 TLV 载荷、配网/OTA 扩展子协议字节布局须在量产前完成第 19 章验收并同步冻结至 CMD 注册表）  
> **协议版本：** 1.1 (`ProtocolVersion = 0x0101`)  
> **前向兼容策略：** V1.1 相对 V1.0 存在帧格式变更，不兼容 V1.0 帧；设备与 APP 必须成套升级或通过独立兼容层识别版本。  
> **适用对象：** Flutter APP（iOS / Android）、BLE 设备固件、Wi-Fi 双模设备、产测工具及后续设备 SDK。  
> **产品范围：** RGB 灯带、RGBIC 分段灯带、智能灯泡、户外装饰灯、太阳能灯、智能灯串等智能照明设备。

---

## 文档说明与规范用语

本文中的“必须 / 不得 / 应 / 建议 / 可以”具有如下含义：

- **必须 / 不得（MUST / MUST NOT）：** 实现互通所必需的强制要求。
- **应 / 不应（SHOULD / SHOULD NOT）：** 除非有明确理由，否则应遵循。
- **建议（RECOMMENDED）：** 推荐实现方式。
- **可以（MAY）：** 可选能力。

本文定义的是项目私有应用层协议，不替代 Bluetooth Core Specification、GATT、ATT、SMP 或 Wi-Fi 标准。

### 版本变更摘要（V1.0 → V1.1）

V1.1 在保留 V1.0 广播识别思路、GATT 服务划分、CRC-16/CCITT-FALSE 和状态驱动原则的基础上，作出以下调整：

1. 将应用帧 `SEQ` 从 1 字节扩展为 2 字节，帧固定开销由 9 字节变为 **10 字节**（保持最大完整帧 512 字节，最大载荷调整为 502 字节）。
2. 明确 REQUEST / RESPONSE / NOTIFY / EVENT 语义及事务匹配规则。
3. 移除“所有 CMD 均以 Length=0 表示 Query、Length>0 表示 Set”的全局约定；每条 CMD 独立定义操作语义。
4. 将 CRC 错误定义为静默丢弃，不再使用无法可靠返回的 `CHECKSUM_ERROR` 应答码（`0x03` 转为保留）。
5. 完善流式组帧、最大帧长、分片串行化、超时、重试、重复请求及幂等规则。
6. 将广播状态序号明确为 8 位模 256 序号，并定义比较边界。
7. 将设备身份、产品能力、外观型号分层；明确 BLE MAC 地址不得作为跨平台永久设备身份。
8. 增加安全会话、绑定、Wi-Fi 配网、RGBIC 分段控制、效果控制、OTA、事件及状态同步规范。
9. 增加版本协商、错误处理、测试向量要求及量产验收清单。

> **重要：** V1.1 的帧头仍为 `0xFE`，但帧布局已经变化。V1.0 的设备不能仅凭帧头区分两个版本。V1.1 设备应通过广播 `AdvVer`（`0x02`）、GATT 固件协议版本或独立产品版本配置确定协议版本，禁止在同一连接中猜测解析两种帧格式。

---

# 1. 总体架构与基本约定

## 1.1 角色模型

| 角色 | BLE 角色 | 职责 |
|---|---|---|
| APP | Central / GATT Client | 扫描、连接、认证、绑定、控制、状态同步、配网、OTA 管理 |
| 智能设备 | Peripheral / GATT Server | 广播、提供 GATT 服务、执行指令、上报状态和事件 |
| 产测工具 | Central / GATT Client | 受控访问设备产测服务和诊断能力 |
| 云端（可选） | 非 BLE 角色 | 设备账户、远程控制、物模型、固件分发及设备管理 |

APP 采用 Flutter/Dart 实现跨平台业务层。iOS 与 Android 的 BLE 扫描、连接、权限、MTU、写入长度及后台行为由平台 BLE API 或原生桥接层适配。业务层不得假设两端底层行为完全一致。

## 1.2 传输架构

协议业务层与传输层解耦：

```text
Flutter UI / Device Domain
          |
Device Capability & State Model
          |
Command Router / Protocol Codec
          |
+----------------------+----------------------+
| BLE Transport        | Wi-Fi / IP Transport |
| GATT Client          | TCP / UDP / MQTT等    |
+----------------------+----------------------+
          |
Smart Lighting Device
```

- BLE 和 Wi-Fi 可以承载相同的设备业务模型，但不要求底层报文完全相同。
- BLE 是近场发现、绑定、配网和本地控制的主要通道。
- Wi-Fi 是联网设备的局域网 / 云端通信通道，具体协议由产品联网方案定义。
- APP 必须通过统一设备状态模型合并不同传输通道的数据。
- 同一设备同时通过 BLE 与 Wi-Fi 在线时，必须定义控制通道优先级和状态版本仲裁，避免旧通道状态覆盖新状态。

## 1.3 字节序

| 数据类型 | 字节序 / 规则 |
|---|---|
| 应用帧多字节整数 | Big-Endian |
| `Length` | Big-Endian |
| `SEQ` | Big-Endian |
| `CRC16` | Big-Endian |
| `BID` / `PID` / `ModelID` | 按各字段定义；多字节整数 Big-Endian |
| Bluetooth SIG Company Identifier | 按 BLE AD 规范 Little-Endian |
| 16-bit Service UUID 广播编码 | Little-Endian |
| MAC 地址 / DeviceID / 随机数 / 密钥 / 哈希 | 按字节数组原样传输，不转换为整数 |
| UTF-8 文本 | UTF-8，无结尾 `\0` |

除非某字段单独声明，不得自行改变字节序。

## 1.4 文本与字符串

- 协议文本统一使用 UTF-8。
- 字符串字段必须定义最大字节长度，不以字符数代替 UTF-8 字节数。
- 不得自动追加 `\0`。
- SSID、密码、设备名称、场景名称等必须分别定义最大长度和合法字符规则。
- 不得将密码、密钥或令牌写入普通日志。

## 1.5 核心通信原则

1. **应用层闭环：** 需要确认的 REQUEST 必须收到匹配的 RESPONSE。GATT Write Response 只代表 ATT 写入过程得到确认，不代表业务执行成功。
2. **设备状态权威：** 设备返回的实际生效状态是本地控制状态的权威来源。APP 可做乐观 UI，但必须在 RESPONSE / NOTIFY / QUERY 后收敛。
3. **链路与业务分层：** ATT 分包、应用帧组包、业务命令是不同层次，不得混为一谈。
4. **安全优先：** CRC 仅用于传输差错检测，不提供身份认证、机密性或防重放。
5. **能力驱动：** APP 根据设备能力模型展示功能，不得仅依赖 ModelID 或硬编码产品名称。
6. **向后兼容：** 未知可选字段、未知能力位及未知事件应按本文规则安全处理；未知必需语义不得猜测执行。

---

# 2. BLE 广播与设备发现

## 2.1 广播类型与长度

V1.1 默认采用 BLE Legacy Advertising。Legacy Advertising 单个 Advertising Data 或 Scan Response 的数据长度上限为 31 字节；二者分别计算，不应将二者简单相加为一个 31 字节数据区。

主广播包应包含：

- Flags；
- 项目 Control Service UUID；
- Manufacturer Specific Data（如设备采用厂商数据发现方案）。

设备名称优先放置在 Scan Response。APP 不得依赖 Scan Response 才能识别设备、判断产品能力或完成设备匹配。

若未来使用 Extended Advertising，必须定义独立广播配置及 APP 兼容策略，不得直接沿用 31 字节偏移表。

## 2.2 V1.1 主广播包布局（Legacy，31 Bytes）

本项目广播模板保持 V1.0 的 31 字节布局，字段定义如下：

| 偏移 | 字段 | 长度 | 编码 | 说明 |
|---|---|---:|---|---|
| 0–2 | Flags AD | 3B | BLE AD | 示例 `02 01 06` |
| 3–6 | 16-bit Service UUID AD | 4B | BLE AD / LE | 示例 `03 03 55 FF`，UUID `0xFF55` |
| 7–30 | Manufacturer Data AD | 24B | BLE AD | `Length=0x17`、`Type=0xFF`、22B payload |

Manufacturer Data 的完整布局：

| 相对偏移（完整广播） | 字段 | 长度 | 规则 |
|---|---|---:|---|
| 7 | AD Length | 1B | `0x17` |
| 8 | AD Type | 1B | `0xFF` |
| 9–10 | Company Identifier | 2B | SIG Company Identifier，小端序；量产使用合法分配值 |
| 11 | AdvVer | 1B | 广播数据结构版本，V1.1 定义为 `0x02` |
| 12–13 | BID | 2B | Big-Endian（`0x0000` 保留） |
| 14–15 | PID | 2B | Big-Endian（`0x0000` 保留） |
| 16 | ModelID | 1B | 产品外观款型（默认 `0x00`） |
| 17 | CAP | 1B | 能力提示位图 |
| 18 | STATE | 1B | 设备即时状态位图 |
| 19 | StateSeq | 1B | 状态变化序号（模 256） |
| 20–25 | DeviceID_Short | 6B | 默认取 `DeviceID[2..7]`；若启用轮换伪名须配套解析机制 |
| 26–30 | RESERVED | 5B | 未定义时必须填 `0x00` |

### 2.2.1 逐字节用途详表（Byte 0 ~ Byte 30，固件抓包调试核对）

| 字节 | 所属字段 | 示例值 | 用途说明 |
|:---:|---|:---:|---|
| `Byte 0` | `Flags` 长度 | `0x02` | Flags AD 段长度，表示后续跟 2 字节（`Byte 1~2`） |
| `Byte 1` | `Flags` 类型 | `0x01` | SIG 标准 AD Type（Flags） |
| `Byte 2` | `Flags` 标志位 | `0x06` | `Bit 1=1` 普通可发现模式；`Bit 2=1` 不支持经典蓝牙（纯 BLE） |
| `Byte 3` | `UUID` 长度 | `0x03` | Service UUID AD 段长度，表示后续跟 3 字节（`Byte 4~6`） |
| `Byte 4` | `UUID` 类型 | `0x03` | SIG 标准 AD Type（Complete List of 16-bit Service UUIDs） |
| `Byte 5` | `Service UUID` 低字节 | `0x55` | 控制服务 UUID `0xFF55` 低字节（SIG 规定广播 UUID 采用小端序） |
| `Byte 6` | `Service UUID` 高字节 | `0xFF` | 控制服务 UUID `0xFF55` 高字节（供 iOS 后台唤醒与硬件过滤） |
| `Byte 7` | `AD Length` | `0x17` | 厂商数据段长度（十进制 23 = `Byte 8` 1B 类型 + `Byte 9~30` 22B 载荷） |
| `Byte 8` | `AD Type` | `0xFF` | SIG 标准 AD Type（Manufacturer Specific Data） |
| `Byte 9` | `CID` 低字节 | `0xFF` | SIG 厂商识别码低字节（小端序；内测占位 `0xFFFF`，量产用合法分配值） |
| `Byte 10` | `CID` 高字节 | `0xFF` | SIG 厂商识别码高字节 |
| `Byte 11` | `AdvVer` | `0x02` | 广播结构版本号；V1.1 固定 `0x02`，APP 先读该字节决定解析版本 |
| `Byte 12` | `BID` 高字节 | `0x00` | 品牌/业务域识别码高字节（大端序），非本品类 APP 直接忽略 |
| `Byte 13` | `BID` 低字节 | `0x01` | 品牌/业务域识别码低字节（`0x0000` 保留） |
| `Byte 14` | `PID` 高字节 | `0x00` | 产品功能平台码高字节（大端序），决定硬件能力与指令集边界 |
| `Byte 15` | `PID` 低字节 | `0x01` | 产品功能平台码低字节（`0x0000` 保留），同 PID 共享固件语义 |
| `Byte 16` | `ModelID` | `0x00` | 同 `(BID, PID)` 下外观款型（默认 `0x00`，未知款型回退默认 UI） |
| `Byte 17` | `CAP` | Bitmap | 能力提示位图（Bit 0~6，见 2.3） |
| `Byte 18` | `STATE` | Bitmap | 即时状态位图（Bit 0~5，见 2.4） |
| `Byte 19` | `StateSeq` | `0x00~0xFF` | 状态变化模 256 递增序号（见 2.5） |
| `Byte 20–25` | `DeviceID_Short[0..5]` | `DeviceID[2..7]` | 6 字节设备短标识（原始字节流，非认证凭据） |
| `Byte 26–30` | `RESERVED[0..4]` | `0x00` | 预留扩展 5 字节，未定义时必须填 `0x00` |

### 2.2.2 Company Identifier

- `0xFFFF` 仅可用于内部联调占位，不得作为量产厂商身份。
- 量产设备必须使用合法分配给相关制造商的 Bluetooth SIG Company Identifier，或按最终合规方案确定 Manufacturer Data 的使用方式。
- APP 必须保留原始 Manufacturer Data 字节，不能只保存解析后的字段。
- Android 按 Company Identifier 查询 Manufacturer Data 时，平台返回值通常不包含 Company Identifier 本身（从 `Byte 11 AdvVer` 开始）；iOS 的 `CBAdvertisementDataManufacturerDataKey` 通常包含 Company Identifier 字节（从 `Byte 9 CID` 开始）。APP 跨平台解析层必须统一成“原始厂商载荷”抽象，并用抓包验证实际平台返回格式。

### 2.2.3 AdvVer

| AdvVer | 定义 |
|---|---|
| `0x01` | V1.0 广播结构 |
| `0x02` | V1.1 广播结构 |
| 其他 | 未知版本 |

APP 对未知 `AdvVer` 不得按 V1.1 偏移强行解析。可以保留原始广播并提示不支持，或按已注册的兼容解析器处理。

### 2.2.4 Service UUID

项目 Control Service 使用：

`0000FF55-0000-1000-8000-00805F9B34FB`

Legacy 广播中的 16-bit UUID 编码示例为 `55 FF`。

广播该 UUID 可用于扫描过滤及特定 iOS 后台扫描配置，但**不能保证** APP 在所有后台、系统省电、权限或射频环境下必然被唤醒或发现。APP 必须处理扫描延迟、广播丢失、系统合并及后台限制。

## 2.3 CAP 能力位图

CAP 用于扫描阶段快速判断关键通信能力，不代表设备完整功能列表。

| Bit | 名称 | 定义 |
|---:|---|---|
| 0 | BLE_CONTROL | 支持 BLE GATT 控制 |
| 1 | WIFI_PROVISIONING | 支持 Wi-Fi 配网 |
| 2 | BLE_MESH | 支持 BLE Mesh（若产品实际实现） |
| 3 | APP_AUTH_REQUIRED | 控制或敏感操作需要应用层认证 |
| 4 | OTA_SUPPORTED | 支持通过 BLE 执行 OTA |
| 5 | MULTI_CHANNEL | 支持多通道灯光能力 |
| 6 | SEGMENT_CONTROL | 支持分段 / 像素级控制 |
| 7 | RESERVED | 保留，发送 0 |

CAP 位表示设备具备对应能力，不代表当前状态。例如 `OTA_SUPPORTED` 不代表设备正在 OTA。

## 2.4 STATE 状态位图

| Bit | 名称 | 定义 |
|---:|---|---|
| 0 | POWER_ON | 主灯当前开关状态提示 |
| 1 | BOUND | 是否已完成用户绑定 |
| 2 | PAIRING_WINDOW | 是否处于允许绑定的窗口 |
| 3 | OTA_MODE | 是否处于 OTA / Bootloader 模式 |
| 4 | WIFI_CONNECTED | Wi-Fi 当前连接状态 |
| 5 | BUSY | 设备当前是否处于不可接受部分操作的忙碌状态 |
| 6–7 | RESERVED | 保留，发送 0 |

广播 STATE 仅是扫描列表的即时提示。APP 建立连接后必须通过 GATT 查询权威状态。状态位图不应承载复杂错误原因或完整联网状态。

## 2.5 StateSeq 规则

`StateSeq` 为 8 位无符号循环序号。

- 设备每当广播中任一 STATE 位发生变化时，必须递增一次。
- 同一次状态变化产生的同一逻辑广播快照使用相同 StateSeq。
- 普通周期性重复广播不递增。
- 设备重启后可以从任意初值开始；APP 检测到设备重启或广播长时间中断后，必须重置本地序号比较基线。
- APP 使用模 256 比较：

```text
delta = (new_seq - old_seq) mod 256

delta == 0          -> 相同版本
1 <= delta <= 127   -> 新版本
128 <= delta <= 255 -> 旧版本或无法判定
```

该算法仅在两个可比较状态之间的真实变化跨度小于 128 时有效。若间隔过久、可能丢失 128 次以上变化或设备重启，APP 必须重新连接查询，不得仅凭序号判断。

## 2.6 DeviceID 与 DeviceID_Short

### DeviceID

- `DeviceID` 是设备逻辑身份，长度 8 字节。
- 由制造流程生成并写入设备非易失存储。
- 在正常固件升级、OTA、重启、BLE 地址变化后保持不变。
- 恢复出厂是否保留 DeviceID 必须由产品身份策略决定；默认应保留制造身份，不得因用户解绑而重新生成。
- DeviceID 不是密钥，不可单独作为授权凭证。

### DeviceID_Short

- 长度 6 字节，用于附近设备列表消歧和扫描阶段临时关联。
- **默认基线映射**：取 GATT `0x01` 返回的 8 字节 `DeviceID` 末尾 6 字节（`DeviceID[2..7]`）按原始字节序填入（或由出厂标识经不可逆派生生成）。
- 不得将 BLE MAC 地址直接作为 DeviceID。
- 不得使用固定单字节 XOR（如 `raw_mac ^ 0xAD`）等可逆混淆方式代替隐私保护。
- 若高隐私产品线启用轮换标识或基于密钥的短期伪名，必须在产品安全 profile 中冻结伪名解析算法，确保已绑定 APP 在扫描阶段仍可关联设备。
- APP 必须在连接后通过 GATT 读取完整 DeviceID，并完成设备身份一致性校验。
- 广播短标识不得作为绑定或控制授权的唯一依据。

### BLE 地址与跨平台绑定

BLE 地址可能是 Public、Random Static、Resolvable Private Address 或 Non-resolvable Private Address。iOS 不向普通 APP 提供可依赖的设备 MAC 地址；Android 也不应将 MAC 作为跨平台永久身份。

因此：
- 绑定记录必须以 DeviceID（及认证结果）为核心。
- 不得要求 iOS 将扫描到的 MAC 保存后供 Android 连接。
- 跨手机、跨平台重新连接应重新扫描、识别 DeviceID、认证并连接当前广播地址。
- 若设备采用隐私地址，固件必须确保扫描与连接流程符合 BLE 隐私机制及产品配对策略。

## 2.7 Scan Response 设备名称

推荐格式：

`SL_<ModelAbbr>_<4位短ID>`

规则：
- AD Type 使用 `0x09` Complete Local Name，或在长度不足时使用 `0x08` Shortened Local Name。
- UTF-8 编码，不追加 `\0`，总长不超过 29 字节。
- 名称仅用于用户展示，不作为设备身份凭据。
- APP 不得因名称缺失而丢弃已通过广播结构识别的设备。

## 2.8 广播间隔建议

| 状态 | 建议广播间隔 | 说明 |
|---|---:|---|
| 未绑定 / 配网窗口 | 100–200 ms | 优先发现体验；受地区法规及功耗约束 |
| 已绑定、可连接 | 500–1000 ms | 平衡功耗与重连体验 |
| Wi-Fi 在线且产品策略允许 | 500–1000 ms 或关闭可连接广播 | 必须保留用户触发的近场恢复路径 |
| OTA / Bootloader | 按 OTA 方案定义 | 必须保持相同的 `BID`、`PID` 与 `DeviceID_Short`，保证身份可关联 |

广播间隔为建议值，固件最终参数必须结合芯片、天线、Wi-Fi/BLE 共存、功耗和认证测试确定。

---

# 3. GATT 服务与连接流程

## 3.1 UUID 分配

### Control Service

| 项目 | UUID | 属性 |
|---|---|---|
| Control Service | `0000FF55-0000-1000-8000-00805F9B34FB` | Primary Service |
| Write Characteristic | `A83B0002-7D4E-4F9A-9C12-8E4B01000001` | Write、Write Without Response |
| Notify Characteristic | `A83B0003-7D4E-4F9A-9C12-8E4B01000001` | Notify |
| CCCD | `00002902-0000-1000-8000-00805F9B34FB` | Descriptor |

UUID 必须由项目统一注册管理。`0xFF55` 属于项目使用的 16-bit UUID 方案，不应宣称全球唯一。若未来与其他产品或生态发生冲突，应通过新的 128-bit Service UUID 或独立服务版本迁移。

## 3.2 连接初始化顺序

APP 应按以下顺序初始化连接：

1. 扫描并解析广播，记录原始广播数据及候选设备信息。
2. 使用当前扫描结果提供的连接标识发起连接，不使用历史 MAC 作为跨平台永久标识。
3. 发现 GATT 服务和特征。
4. 校验 Control Service、Write Characteristic、Notify Characteristic 是否存在及属性是否正确。
5. 启用 Notify，写入 CCCD `0x0001`，等待订阅操作成功。
6. 可选执行 MTU 协商（如 Android 请求 `247`）；失败不得阻断基本通信。
7. 查询设备基础信息 `0x01`。
8. 核对完整 DeviceID、BID、PID、ModelID 与扫描阶段信息。
9. 根据 CAP / Device Info 决定是否执行安全认证。
10. 查询 Capability Profile（`0x02`）。
11. 查询设备当前状态及灯光参数。
12. 进入 READY 状态，开放业务控制。

在 Notify 未成功订阅前，APP 不得发送依赖 Notify RESPONSE 的业务请求。

## 3.3 MTU 与单次写入长度

- BLE ATT 默认 MTU 通常为 23。
- APP 可以请求较大 MTU，但不能假设操作系统一定接受目标值。
- 实际单次 ATT Value 可承载长度为 `ATT_MTU - 3`。
- 对 Write Request / Write Command，APP 必须遵循平台 API 返回的最大写入长度。
- iOS 与 Android 的 MTU 协商和可写长度暴露方式不同，协议层不得依赖某个固定 MTU。
- V1.1 最大应用帧为 512 字节（含 10B 固定开销及最多 502B Command Data），超出必须使用业务层分块方案或拒绝。

示例：

| ATT MTU | 单次 ATT Value 最大长度 | 单包可容纳最大 Command Data (`ATT_MTU - 3 - 10`) |
|---:|---:|---:|
| 23 | 20 Bytes | 10 Bytes |
| 185 | 182 Bytes | 172 Bytes |
| 247 | 244 Bytes | 234 Bytes |
| 517 | 514 Bytes | 502 Bytes（受单帧上限约束） |

> 注意：应用帧固定开销为 **10 字节**。因此在 `ATT_MTU = 247` 时，单个 GATT PDU 可容纳的 Command Data 最大值为 **234 字节**（`244 - 10`）。

## 3.4 Write / Write Without Response 使用规范

| 场景 | 推荐方式 | 说明 |
|---|---|---|
| 设备信息查询 | Write | 需要业务 RESPONSE |
| 开关、亮度、色温普通设置 | Write | 需要最终结果确认 |
| 滑条拖动中间值 | Write Without Response | 节流发送，不逐帧重试 |
| 滑条松手最终值 | Write | 最终目标值闭环确认 |
| 安全认证、绑定 | Write | 需要业务结果 |
| Wi-Fi 凭证 | Write | 必须安全会话 |
| OTA 控制命令 | Write | 使用 OTA 状态机 |
| OTA 数据块 | 专用 OTA 数据通道 / 按 OTA 规范 | 不套用普通业务重试策略 |

Write Without Response 不代表业务成功，也不保证应用层接收端已执行。需要可靠结果的操作必须使用应用层确认或后续状态查询。

## 3.5 发送队列与并发

- 同一连接必须使用统一发送调度器。
- 同一应用帧的 ATT 分片必须连续发送，不得与其他帧分片交错。
- 多个业务线程不得直接竞争写同一特征。
- 普通控制 REQUEST 同时在途数量建议不超过 4。
- 安全、绑定、配网、OTA、复位等关键流程必须串行执行。
- 设备队列满时返回 `BUSY`；APP 应按退避策略重试，不得立即高频重发。

## 3.6 断线与重连

断线时 APP 必须：
- 停止当前连接的发送任务；
- 清理该连接的未完成事务；
- 丢弃未完成的分片帧；
- 将设备连接状态更新为 DISCONNECTED；
- 不把未确认的控制结果视为成功。

重连后必须：
- 重新发现服务并订阅 Notify；
- 重新读取设备身份；
- 重新执行必要的安全会话认证；
- 查询当前设备状态；
- 不复用旧连接的 SEQ、会话密钥、Nonce 或未完成事务。

---

# 4. 应用层通用数据帧 V1.1

## 4.1 帧结构

V1.1 应用帧固定开销为 **10 字节**。

| 字段 | Head | Type | CMD | Status | SEQ | Length | Command Data | CRC16 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 长度 | 1B | 1B | 1B | 1B | 2B | 2B | N Bytes | 2B |
| 示例 | `FE` | `01` | `20` | `00` | `00 01` | `00 01` | `01` | `xxxx` |

固定字段：
- Head：`0xFE`
- Type：1 Byte
- CMD：1 Byte
- Status：1 Byte
- SEQ：2 Bytes，Big-Endian
- Length：2 Bytes，Big-Endian
- Command Data：0–502 Bytes
- CRC16：2 Bytes，Big-Endian

最大完整帧长度：512 Bytes。  
最大 Command Data：502 Bytes。

## 4.2 字段定义

| 偏移 | 字段 | 长度 | 说明 |
|---:|---|---:|---|
| 0 | Head | 1B | 固定 `0xFE` |
| 1 | Type | 1B | 消息类型 |
| 2 | CMD | 1B | 指令码 |
| 3 | Status | 1B | 状态码 |
| 4–5 | SEQ | 2B | 事务序列号，大端序 |
| 6–7 | Length | 2B | Command Data 长度，大端序 |
| 8 至 7+N | Command Data | N | 业务载荷 |
| 8+N 至 9+N | CRC16 | 2B | 覆盖 Head 至 Command Data |

## 4.3 Message Type

| Type | 名称 | 方向 | 说明 |
|---:|---|---|---|
| `0x01` | REQUEST | APP → Device | APP 发起的业务请求 |
| `0x02` | RESPONSE | Device → APP | 对 REQUEST 的匹配响应 |
| `0x03` | NOTIFY | Device → APP | 设备当前状态同步 |
| `0x04` | EVENT | Device → APP | 一次性事件或异步操作事件 |
| `0x05`–`0x7F` | RESERVED | — | 保留 |
| `0x80`–`0xFF` | PRIVATE | — | 未经注册不得使用 |

### REQUEST

- Status 必须为 `0x00`。
- SEQ 必须为 `0x0001`–`0xFFFF`。
- 每个需要闭环的 REQUEST 必须对应一个 RESPONSE。
- 每条 CMD 自行定义 Request Data 结构，不得全局以 `Length=0` 判定 Query。

### RESPONSE

- SEQ 必须与对应 REQUEST 一致。
- CMD 必须与对应 REQUEST 一致。
- Status 表示业务执行结果。
- 成功时按 CMD 定义返回数据；失败时按状态码规则返回错误数据。
- 一个 REQUEST 只能有一个最终 RESPONSE。长时间异步任务应先返回已接受状态（`Status=0x00` 并携带 `TaskID` 或接受标记），再通过 EVENT 上报进度与完成结果，或使用专门的任务查询命令。

### NOTIFY

- SEQ 固定为 `0x0000`。
- Status 固定为 `0x00`。
- 用于同步当前状态快照或状态变化。
- NOTIFY 可被后续状态覆盖；APP 可通过 QUERY 恢复当前状态。
- NOTIFY 不承诺可靠投递。

### EVENT

- SEQ 固定为 `0x0000`。
- Status 固定为 `0x00`；事件结果码放在事件载荷中。
- 表示一次发生的动作或异步任务进展。
- 重要事件应携带事件 ID 或任务 ID，便于 APP 去重和关联。
- V1.1 默认不保证 EVENT 可靠投递。需要可靠投递的事件必须提供确认、重放或可查询的持久化任务状态机制。

## 4.4 SEQ 分配、事务匹配与重用

- SEQ 为 UInt16，范围 `0x0001`–`0xFFFF`；`0x0000` 保留给 NOTIFY / EVENT。
- APP 在单条 BLE 连接内递增分配 SEQ，绕过 `0x0000`。
- 同一连接中，未完成事务的 SEQ 不得重用。
- 断线后旧事务全部失效；新连接可重新从 `0x0001` 开始。
- RESPONSE 必须同时匹配当前连接、SEQ、CMD 和预期 RESPONSE 类型。
- 未匹配 RESPONSE 应记录诊断信息并忽略，不得错误完成其他事务。
- 设备必须在处理请求前检查事务重复性。

### 4.4.1 重复请求与幂等

对使用 Write 重试的 REQUEST，APP 重试时必须复用相同 SEQ 和完全相同的帧内容。

设备端必须维护有限事务缓存：
- 缓存键：当前连接会话 + SEQ；
- 缓存内容：CMD、请求内容摘要、最终 RESPONSE；
- 同一 SEQ 且 CMD / 请求内容一致：不得重复执行，返回缓存 RESPONSE；
- 同一 SEQ 但 CMD 或请求内容不同：返回 `SEQ_CONFLICT`；
- 缓存有效期至少覆盖 APP 最大重试窗口及合理链路延迟；
- 断开连接后可清除连接级缓存，但不得因此重复执行已经持久化的异步任务。

幂等性分类：

| 操作类型 | 例子 | 规则 |
|---|---|---|
| 天然幂等设置 | 设置亮度为 50% | 重复执行结果相同 |
| 非幂等动作 | 切换开关、下一场景 | 不得在不去重情况下自动重试 |
| 高风险动作 | 恢复出厂、删除绑定、OTA 开始 | 必须授权、事务去重及明确结果 |
| 异步任务 | 配网、OTA、校准 | 使用 TaskID，重复请求返回已有任务状态 |

业务指令应优先使用“设置目标值”而不是“切换状态”，以便实现幂等。

## 4.5 Length 与数据合法性

- Length 只表示 Command Data 字节数，不包含帧头或 CRC。
- Length 必须与实际帧长度严格一致。
- 最大 Length 为 502。
- 超过最大值不得分配大缓冲区等待数据。
- 各 CMD 必须定义精确长度、可变长度结构及最大值。
- 保留字段必须按定义填 0；接收端对非零保留字段应按 CMD 规则拒绝或忽略。

## 4.6 CRC-16/CCITT-FALSE

参数：

| 参数 | 值 |
|---|---|
| Polynomial | `0x1021` |
| Initial Value | `0xFFFF` |
| RefIn | False |
| RefOut | False |
| XorOut | `0x0000` |
| Check | `"123456789"` → `0x29B1` |
| CRC 字节序 | Big-Endian |

CRC 覆盖范围：从 Head `0xFE` 到 Command Data 最后一个字节，不包含 CRC 本身。

```c
uint16_t ble_crc16_ccitt_false(const uint8_t *data, size_t len) {
    uint16_t crc = 0xFFFF;
    if (data == NULL && len != 0) {
        return crc;
    }

    for (size_t i = 0; i < len; i++) {
        crc ^= (uint16_t)data[i] << 8;
        for (uint8_t bit = 0; bit < 8; bit++) {
            crc = (crc & 0x8000)
                ? (uint16_t)((crc << 1) ^ 0x1021)
                : (uint16_t)(crc << 1);
        }
    }
    return crc;
}
```

```dart
int crc16CcittFalse(List<int> data) {
  var crc = 0xFFFF;
  for (final value in data) {
    crc ^= (value & 0xFF) << 8;
    for (var bit = 0; bit < 8; bit++) {
      crc = (crc & 0x8000) != 0
          ? ((crc << 1) ^ 0x1021) & 0xFFFF
          : (crc << 1) & 0xFFFF;
    }
  }
  return crc;
}
```

```typescript
export function crc16CcittFalse(data: Uint8Array): number {
  let crc = 0xFFFF;
  for (const value of data) {
    crc ^= value << 8;
    for (let bit = 0; bit < 8; bit++) {
      crc = (crc & 0x8000)
        ? ((crc << 1) ^ 0x1021) & 0xFFFF
        : (crc << 1) & 0xFFFF;
    }
  }
  return crc;
}
```

CRC 不得用作身份认证、加密、签名或防重放机制。

## 4.7 流式解析与 ATT 分片

ATT 数据段边界不等于应用帧边界。接收端必须支持：

1. 一个 ATT Value 中包含一个完整应用帧；
2. 一个应用帧跨多个 ATT Value；
3. 一个 ATT Value 中包含多个完整应用帧；
4. 一个 ATT Value 末尾包含完整帧及下一帧的部分数据。

### 4.7.1 发送规则

- 同一应用帧的 ATT 分片必须按顺序连续发送。
- 同一方向同一连接不允许不同应用帧的分片交错。
- APP 使用统一发送队列串行化写入。
- 对 Write Without Response，发送端必须遵守平台发送缓冲和流控能力；不得无限制快速写入。
- 对 Notify，设备端必须遵守 ATT/GATT 发送队列与连接事件节奏。
- 分片期间若连接断开，丢弃该未完成帧；不得在新连接继续发送旧帧残片。

### 4.7.2 接收状态机

1. **同步帧头：** 丢弃帧头前的无效字节，寻找 `0xFE`。
2. **读取固定头：** 收集前 8 字节。
3. **长度检查：** 读取 Length；若大于 502，丢弃当前候选帧并重新同步。
4. **等待完整帧：** 收集 `10 + Length` 字节。
5. **CRC 校验：** CRC 错误时静默丢弃。
6. **字段检查：** 校验 Type、SEQ、Status 和 CMD 的合法组合：
   - 未知/未注册 `Type`，或设备端收到非 `REQUEST` 帧，或 `REQUEST` 的 `SEQ == 0x0000`（无有效事务序号可回显）：记录诊断日志并丢弃；
   - `REQUEST` 的 `SEQ != 0x0000` 但 `Status != 0x00`：返回 `PARAMETER_ERROR (0x04)` 并回显该 `SEQ` 与 `CMD`。
7. **业务分发：** 交给 CMD 解析器。
8. **连续解析：** 缓冲区仍有数据时继续寻找下一帧。

### 4.7.3 接收缓冲与超时

- 每连接接收缓冲区必须设置固定上限，至少可容纳一个最大应用帧及必要的下一帧残片；实现应避免无界增长。
- 半帧超时建议 1000 ms，具体可按连接参数、传输方式和固件调度调整。
- 超时后丢弃不完整候选帧并重新同步。
- 长度非法、CRC 错误或解析失败不得导致后续合法帧永久阻塞。
- 对恶意或异常输入必须限制 CPU 时间、内存分配和日志频率。

## 4.8 超时与重试

建议默认值（产品可调整，但 APP 与固件必须一致）：

| 操作 | 超时建议 | 重试 |
|---|---:|---|
| 普通查询 / 设置 | 1500 ms | 最多 2 次 |
| 安全握手步骤 | 按握手阶段定义 | 不盲目重试；Nonce 不得复用 |
| 配网提交 | 3–5 s 接受响应；后续用任务状态 | 不重复创建任务 |
| OTA 控制 | 按 OTA 阶段定义 | 由 OTA 协议管理 |
| 高频滑条 WWR 中间帧 | 不等待 RESPONSE | 不重试 |
| 滑条最终值 | 1500 ms | 最多 2 次，复用 SEQ |

- 超时重试必须复用原 SEQ 和完全相同的请求数据。
- 仅对幂等或已实现事务去重的操作自动重试。
- 非幂等动作超时后应先查询状态或任务结果，不得直接再次执行。
- BUSY 应使用指数退避或固定退避，不得立即循环重试。
- 达到最大重试次数后向上层返回明确超时错误。

## 4.9 安全边界

CRC 与 SEQ 不是安全机制。敏感操作必须在认证成功的会话中执行。BLE 链路加密可作为额外保护，但不能自动替代应用层设备身份认证和授权。

---

# 5. 状态码字典

| Status | 常量 | 含义 | 建议行为 |
|---:|---|---|---|
| `0x00` | SUCCESS | 成功 | 返回成功数据 |
| `0x01` | INVALID_COMMAND | CMD 未定义或不支持 | 拒绝执行 |
| `0x02` | LENGTH_ERROR | 载荷长度错误 | 拒绝执行 |
| `0x03` | RESERVED | 保留（原 V1.0 `CHECKSUM_ERROR` 废弃） | 不得分配 |
| `0x04` | PARAMETER_ERROR | 参数非法或越界 | 返回错误 |
| `0x05` | CONDITION_NOT_MET | 当前条件不允许 | 返回错误 |
| `0x06` | BUSY | 设备忙或队列满 | APP 退避 |
| `0x07` | NOT_AUTHORIZED | 未认证或权限不足 | 拒绝执行 |
| `0x08` | RESOURCE_ERROR | 资源不足 | 记录诊断 |
| `0x09` | INTERNAL_ERROR | 内部错误 | 记录诊断 |
| `0x0A` | VERSION_UNSUPPORTED | 协议 / 功能版本不支持 | 协商或升级 |
| `0x0B` | OPERATION_TIMEOUT | 设备内部操作超时 | 查询任务状态 |
| `0x0C` | SEQ_CONFLICT | 相同 SEQ 对应不同请求 | 拒绝执行 |
| `0x0D` | NOT_FOUND | 目标对象不存在 | 检查 ID |
| `0x0E` | ALREADY_EXISTS | 对象已存在 | 返回已有对象信息 |
| `0x0F` | RATE_LIMITED | 请求频率超限 | 延迟后重试 |
| `0x10` | AUTH_FAILED | 认证失败 | 结束认证流程 |
| `0x11` | EXPIRED | 令牌、任务或窗口已过期 | 重新发起流程 |
| `0x12` | UNSUPPORTED_MODE | 当前模式不支持该操作 | 查询能力 |
| `0x13` | INTEGRITY_ERROR | 业务对象完整性验证失败 | 拒绝应用数据 |
| `0x14`–`0x7F` | RESERVED | 保留 | 注册后使用 |
| `0x80`–`0xFF` | VENDOR | 厂商私有 | 必须登记 |

**CRC 错误不返回 Status。** CRC 错误帧必须静默丢弃，因为帧内容本身不可信，无法可靠确认 CMD 和 SEQ。

---

# 6. 设备身份、信息与能力模型

## 6.1 身份字段

| 字段 | 长度 | 说明 |
|---|---:|---|
| BID | 2B | 品牌 / 产品业务域标识 |
| PID | 2B | 产品功能平台标识 |
| ModelID | 1B | 同一 PID 下的外观 / 款型标识 |
| DeviceID | 8B | 设备唯一逻辑身份 |
| HardwareRevision | 2B | 硬件版本 |
| FirmwareVersion | 4B | 固件版本编码 |
| ProtocolVersion | 2B | 主协议版本，例如 `0x0101` |
| BootloaderVersion | 4B | 完整 Bootloader 版本（基础信息 `0x01` 返回 2B 主次版本，完整 4B 通过扩展诊断 CMD 查询） |
| CapabilityVersion | 1B | 能力结构版本 |

### 6.1.1 BID / PID / ModelID 规则

- BID 标识品牌或业务域。
- PID 标识功能平台、硬件能力和指令集兼容边界。
- ModelID 只决定外观、资源和 UI 差异，不得改变 PID 已定义的指令语义。
- 已正式发布的 PID 不得被重新赋予不同协议语义。
- 新外观但硬件能力和协议完全兼容时，可以使用新 ModelID。
- 新硬件能力或不同指令语义应使用新 PID，或通过能力模型明确扩展且保持兼容。
- 未知 ModelID 应回退默认 UI（`0x00` 款型），不得阻断配网或基础控制。
- 未知 PID 应尝试加载已注册或云端动态能力描述；无法理解必需能力时提示升级，不得猜测控制。

PID 注册表由项目统一维护，不在固件中随意分配。

## 6.2 CMD `0x01`：设备基础信息

**Request：** 空载荷，`Length=0`。

**Response Data（固定 24 Bytes）：**

| Offset | 字段 | 长度 | 说明 |
|---:|---|---:|---|
| 0–1 | ProtocolVersion | 2B | `0x0101` 表示 V1.1 |
| 2–3 | HardwareRevision | 2B | 大端 |
| 4–7 | FirmwareVersion | 4B | `Major.Minor.Patch.Build` 编码规则由固件版本规范定义 |
| 8–15 | DeviceID | 8B | 唯一逻辑身份 |
| 16–17 | BID | 2B | 大端 |
| 18–19 | PID | 2B | 大端 |
| 20 | ModelID | 1B | 款型 |
| 21 | CapabilityVersion | 1B | 能力结构版本 |
| 22–23 | BootloaderVersionMajorMinor | 2B | Bootloader 主次版本；完整 4B 版本通过扩展诊断信息 CMD 查询 |

如需完整 Bootloader 版本、制造批次、生产日期、序列号等，应通过后续设备诊断信息 CMD 获取，不得不断改变本固定结构。

## 6.3 CMD `0x02`：设备能力查询

**Request：** 空载荷，`Length=0`。

**Response Data：**

| 字段 | 长度 | 说明 |
|---|---:|---|
| CapabilityVersion | 1B | 能力结构版本 |
| TotalLength | 2B | 后续能力 TLV 总长度（Big-Endian） |
| Capability TLV | N | TLV 能力列表 |

TLV 格式：

```text
Tag: 1 Byte
Length: 2 Bytes (Big-Endian)
Value: Length Bytes
```

未知 Tag 必须跳过；重复 Tag 的处理方式由 Tag 定义。必需能力缺失时 APP 不得猜测。

建议能力 Tag（各 Tag 的精确 Value 字节布局须在 CMD 注册表与 Capability Profile 规范中冻结）：

| Tag | 名称 | Value 语义 |
|---:|---|---|
| `0x01` | LIGHT_CHANNELS | 通道数量及通道类型 |
| `0x02` | COLOR_MODES | 支持的颜色模式列表 |
| `0x03` | BRIGHTNESS_RANGE | 最小 / 最大 / 步进 |
| `0x04` | CCT_RANGE | 最小 / 最大 Kelvin |
| `0x05` | RGB_FORMAT | RGB / RGBW / RGB+CCT 格式 |
| `0x06` | SEGMENT_INFO | 分段数量、寻址范围、最大批量数（见 9.2） |
| `0x07` | EFFECT_CAPABILITY | 内置效果 ID / 参数版本 |
| `0x08` | WIFI_CAPABILITY | Wi-Fi 频段、配网能力 |
| `0x09` | OTA_CAPABILITY | 支持的 OTA 通道和块大小 |
| `0x0A` | SECURITY_CAPABILITY | 认证算法套件与安全版本 |
| `0x0B` | POWER_CAPABILITY | 电源、调光及功率限制能力 |

## 6.4 能力与物模型原则

- 广播 CAP 是快速提示；GATT Capability 是连接后的权威能力描述。
- APP 应将能力映射为统一 Device Profile。
- 业务 UI 不得直接以 MAC、设备名称或 ModelID 判断是否显示 RGBIC 控件。
- 云端 TSL 可以扩展云端业务属性，但不能覆盖本地协议未声明的硬件能力。
- 设备不支持的 CMD 必须返回 `INVALID_COMMAND` 或 `UNSUPPORTED_MODE`，不得静默执行近似行为。

---

# 7. 安全认证、绑定与会话

## 7.1 安全目标

安全设计必须覆盖：

- 设备身份真实性；
- APP / 用户绑定授权；
- 会话密钥协商；
- 敏感数据机密性；
- 消息完整性；
- 重放防护；
- 解绑和恢复出厂授权；
- 配网凭证保护；
- OTA 固件真实性。

## 7.2 安全原则

1. DeviceID、广播短标识、BLE MAC、Local Name 均不是密钥。
2. CRC16 不提供密码学完整性。
3. 不得将固定共享密码写入所有设备固件。
4. 不得在广播中发送 Wi-Fi 密码、令牌、密钥或认证材料。
5. 每台设备应具有唯一设备密钥或等效安全身份材料；优先存储在安全硬件或受保护区域。
6. 认证应使用经过审查的标准密码学协议和成熟密码库，不自行设计加密算法。
7. 若产品尚未确定安全算法，必须在量产前冻结算法套件、密钥生命周期和工厂注入流程。

## 7.3 安全握手接口

V1.1 预留以下系统 CMD：

| CMD | 名称 | 说明 |
|---:|---|---|
| `0x03` | SECURITY_CAPABILITY | 查询安全能力 / 算法套件 |
| `0x04` | AUTH_START | 开始认证 |
| `0x05` | AUTH_CHALLENGE | 交换挑战数据 |
| `0x06` | AUTH_FINISH | 完成双向认证并建立会话 |
| `0x07` | SESSION_CLOSE | 主动关闭会话 |
| `0x08` | BIND_DEVICE | 请求绑定 |
| `0x09` | UNBIND_DEVICE | 请求解绑 |

上述 CMD 仅定义接口编号和流程职责；具体 Data 字段必须由选定的标准认证方案配置文件进一步冻结。未完成密码学评审前，不得将其视为可量产安全实现。

## 7.4 会话安全要求

认证成功后建立会话上下文，至少包含：
- SessionID；
- 会话密钥或安全通道句柄；
- 双方随机数 / Nonce 状态；
- 会话有效期；
- 权限范围；
- 计数器或防重放状态。

敏感业务载荷必须通过经过认证的加密与完整性保护。可采用标准 AEAD（例如 AES-CCM 或 AES-GCM，具体选择需与 MCU、平台和密码库能力匹配），并明确 Nonce 唯一性规则。

如果使用应用层加密封装，必须定义：
- 算法套件 ID；
- Key Derivation；
- Nonce 格式；
- AAD 字段；
- 密文长度；
- Tag 长度；
- 计数器窗口；
- 会话重建；
- 认证失败后的清理行为。

不得在未完成这些定义时仅通过 `CAP.Bit3` 宣称安全功能已实现。

## 7.5 绑定规则

- 绑定必须发生在明确的用户操作和设备可绑定窗口内。
- APP 必须向用户展示待绑定设备的可识别信息，降低误绑风险。
- 广播 `BOUND=0` 且 `PAIRING_WINDOW=1` 只是候选条件，最终必须由设备端授权确认。
- 设备端必须限制绑定窗口时长及失败尝试次数。
- 绑定成功后，设备应持久化授权关系，并通过 NOTIFY / QUERY 更新状态。
- 解绑必须验证当前授权；不能仅凭 DeviceID 执行。
- 多用户共享、家庭成员权限和云端账户关联由上层账户系统定义。

## 7.6 恢复出厂安全规则

恢复出厂必须是授权操作。设备收到请求后：
1. 校验会话权限；
2. 校验设备当前状态；
3. 创建复位任务；
4. 返回任务已接受；
5. 执行清理；
6. 按产品策略断开连接并重启；
7. 重新广播未绑定状态。

不得用固定延时作为 APP 已收到 RESPONSE 的保证。APP 应通过断线、重新扫描、DeviceID 核验和状态查询确认复位结果。

---

# 8. 通用控制指令

## 8.1 CMD 号段规划

| 区间 | 类别 | 用途 |
|---|---|---|
| `0x00` | 保留 | 禁止使用 |
| `0x01`–`0x1F` | 系统 / 身份 / 安全 | 信息、能力、认证、绑定、复位、时钟 |
| `0x20`–`0x3F` | 基础控制 | 电源、亮度、渐变、设备状态 |
| `0x40`–`0x5F` | 颜色与光学 | CCT、RGB、RGBW、颜色模式 |
| `0x60`–`0x7F` | 场景 / 效果 / 事件 | 场景、动态效果、按键、传感 |
| `0x80`–`0x9F` | 分段 / 像素控制 | RGBIC、灯珠、分区 |
| `0xA0`–`0xBF` | Wi-Fi 配网 | 网络扫描、提交、状态、清除 |
| `0xC0`–`0xDF` | OTA | 固件更新 |
| `0xE0`–`0xFE` | 产测 / 诊断 | 工厂测试，量产必须权限锁定 |
| `0xFF` | 保留 | 禁止使用 |

## 8.2 通用 CMD 定义模板

每条 CMD 必须定义：

- CMD 编号和名称；
- 允许的 Message Type；
- 请求 Data 格式及精确长度（含 QUERY 与 SET 各自布局）；
- 成功 RESPONSE Data；
- 错误 Status；
- 是否要求认证；
- 幂等性；
- 超时与重试策略；
- 是否产生 NOTIFY / EVENT；
- 适用 PID / 能力；
- 协议版本要求；
- 至少一个正向和一个异常测试向量。

## 8.3 CMD `0x11`：恢复出厂

Request：空载荷（`Length=0`）。  
要求：已认证且具备管理权限。

Response：建议返回 `TaskID (UInt32, BE)`，表示设备已接受复位任务；Status SUCCESS 不代表清理已全部完成。

设备完成后：
- 清除用户网络凭证、绑定授权和用户配置；
- 保留 DeviceID、制造校准和必要的安全身份材料；
- 按产品安全策略轮换会话 / 配对材料；
- 断开连接并重新进入初始广播状态。

具体保留与清除数据必须在产品数据清单中列明。

## 8.4 CMD `0x20`：电源控制

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `PowerState (1B)`：`0x00=OFF`、`0x01=ON`。

长度：
- QUERY：1B；
- SET：2B。

Response / NOTIFY Data（1B）：
- `PowerState (1B)`：设备实际开关状态（`0x00=OFF`、`0x01=ON`）。

该结构避免将空载荷同时解释为 Query 与 Action。

## 8.5 CMD `0x21`：主亮度控制

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `Brightness (1B)`，范围 `0–100`。

长度：
- QUERY：1B；
- SET：2B。

Response / NOTIFY Data（1B）：
- `Brightness (1B)`：设备实际生效值（`0–100`）。

设备可因温度、功率或硬件限制进行限幅，必须返回实际值。

## 8.6 CMD `0x40`：色温控制

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `CCT_Kelvin (UInt16, BE)`。

长度：
- QUERY：1B；
- SET：3B。

范围由 Capability Profile 指定。APP 不得假设所有设备均支持相同色温范围。

Response / NOTIFY Data（2B）：
- `CCT_Kelvin (UInt16, BE)`：设备实际生效色温值。

## 8.7 CMD `0x41`：颜色模式

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `ColorMode (1B)`。

长度：
- QUERY：1B；
- SET：2B。

Response / NOTIFY Data（1B）：
- `ColorMode (1B)`：设备当前实际生效颜色模式。

ColorMode 建议注册：

| 值 | 模式 |
|---:|---|
| `0x00` | WHITE / 单色白光 |
| `0x01` | CCT |
| `0x02` | RGB |
| `0x03` | RGBW |
| `0x04` | RGB+CCT |
| `0x05` | EFFECT |
| `0x06` | SEGMENT |
| 其他 | 保留 |

设备不支持的模式返回 `UNSUPPORTED_MODE`。

## 8.8 CMD `0x42`：RGB 颜色

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `R (1B), G (1B), B (1B)`。

长度：
- QUERY：1B；
- SET：4B。

Response / NOTIFY Data（3B）：
- `R (1B), G (1B), B (1B)`（各通道范围 `0–255`）。若设备内部色彩空间不同，设备负责转换并返回实际值或标准化状态。

## 8.9 CMD `0x43`：RGBW / RGB+CCT 扩展颜色

按 Capability Profile 中声明的通道类型和顺序编码。不得在同一 PID 内无版本说明地改变通道顺序。

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- QUERY 时仅含 `Operation (1B)`（或按 `ChannelType` 扩展，须在 CMD 注册表冻结）；
- SET 时追加以下 TLV 结构：

```text
ChannelType (1B)
ChannelCount (1B)
ChannelValues (ChannelCount Bytes)
```

Response / NOTIFY Data：
- `ChannelType (1B) + ChannelCount (1B) + ChannelValues (ChannelCount Bytes)`。

通道类型枚举必须在产品能力注册表中统一冻结定义。

## 8.10 CMD `0x44`：渐变 / 转换参数

Request：
- `Operation (1B)`：`0x00=QUERY`、`0x01=SET`；
- SET 时追加 `TransitionTimeMs (UInt16, BE)` + `EasingType (UInt8)`（共 4B）。

Response / NOTIFY Data（3B）：
- `TransitionTimeMs (UInt16, BE)` + `EasingType (UInt8)`。

TransitionTime 最大值及 `EasingType` 枚举由产品能力注册表定义。未声明支持渐变的设备必须返回 `UNSUPPORTED_MODE`。

## 8.11 高频滑条策略

- 拖动过程中使用 Write Without Response，建议 50–100 ms 节流。
- 允许中间值被后续值覆盖。
- 中间帧不进入普通超时重试队列。
- 松手后发送最终目标值的 Write Request，并等待 RESPONSE。
- 最终 RESPONSE 返回实际生效值。
- 若最终确认失败，APP 查询设备实际状态并修正 UI。
- 同一控制属性最多一个最终确认请求在途。
- 不能将“滑条中间帧已写入 ATT”当作设备已完成执行。

---

# 9. RGBIC 与分段控制协议

## 9.1 设计目标

支持 RGBIC 灯带、户外装饰灯、可寻址灯串等具备独立分段或像素控制能力的产品。

协议必须区分：
- 物理 LED 数量；
- 逻辑 Segment 数量；
- APP 展示分区；
- 可寻址像素范围；
- 控制分辨率；
- 内置效果与逐段自定义效果。

## 9.2 Segment 能力信息

Capability `SEGMENT_INFO`（Tag `0x06`）至少应提供：

| 字段 | 长度 | 说明 |
|---|---:|---|
| SegmentCount | 2B | 当前逻辑分段数（Big-Endian） |
| AddressUnit | 1B | `0=Segment`、`1=Pixel` |
| MaxBatchItems | 2B | 单条指令最大控制项数量（Big-Endian） |
| IndexBase | 1B | 起始索引，统一建议 `0` |
| ColorFormat | 1B | RGB / RGBW 等（与 CMD `0x81` 枚举对齐） |
| Flags | 2B | 是否支持单段亮度、反向、分组等（Big-Endian） |

## 9.3 CMD `0x80`：分段信息查询

Request：空载荷（`Length=0`）。  
Response：返回 `SegmentCount`、寻址方式、最大批量数、颜色格式及能力标志（与 9.2 字段对齐）。

## 9.4 CMD `0x81`：分段颜色设置

Request 结构：

| 字段 | 长度 | 说明 |
|---|---:|---|
| Operation | 1B | `0x00=QUERY`、`0x01=SET` |
| StartIndex | 2B | 起始 Segment / Pixel（Big-Endian） |
| ItemCount | 2B | 查询或设置的项目数量（Big-Endian） |
| ColorFormat | 1B | SET 时必填颜色格式；QUERY 时填请求格式或由设备按能力格式返回 |
| Values | N | 仅 SET 时携带，每个项目的颜色数据；QUERY 时 `N=0` |

Response / NOTIFY Data：
- `StartIndex (2B, BE) + ItemCount (2B, BE) + ColorFormat (1B) + Values (N Bytes)`。

每个颜色项的长度由 `ColorFormat` 决定。总长度不得超过单帧最大载荷（502 Bytes）；超长批量数据必须采用应用层分块事务（CMD `0x84`）。

## 9.5 CMD `0x82`：分段亮度设置

按 `Operation (1B)` + `StartIndex (2B, BE)` + `ItemCount (2B, BE)` + `BrightnessValues (SET 时为 ItemCount Bytes，QUERY 时省略)` 编码；Response 返回实际 `StartIndex + ItemCount + BrightnessValues`。设备必须校验索引范围及批量数量。

## 9.6 CMD `0x83`：分段效果设置

建议字段（具体字节布局与枚举在 CMD 注册表冻结）：
- SegmentIndex / SegmentRange；
- EffectID；
- EffectVersion；
- Speed；
- Direction；
- Intensity；
- EffectParameters TLV。

效果 ID 必须由项目效果注册表统一维护。APP 不得假设不同 PID 的同一 EffectID 参数结构完全一致，除非注册表明确保证兼容。

## 9.7 CMD `0x84`：分段数据块写入

用于大规模分段 / 像素数据更新，必须使用应用层分块机制，包含（具体字段宽度在 CMD 注册表冻结）：
- TransferID；
- BlockIndex；
- TotalBlocks 或总字节数；
- Payload；
- 可选块校验。

设备必须提供接收窗口、确认、超时、取消和失败恢复规则。不得将普通 GATT ATT 分片误认为业务层 Transfer Block。

---

# 10. 场景与动态效果

## 10.1 场景模型

场景应具有稳定 SceneID。建议支持：
- 查询场景列表；
- 应用场景；
- 创建 / 更新用户场景（设备支持时）；
- 删除用户场景；
- 查询当前场景。

场景可以由设备内置或 APP / 云端管理。设备端存储能力必须通过 Capability 声明。

## 10.2 CMD 号段建议

| CMD | 功能 |
|---:|---|
| `0x60` | 查询当前场景 |
| `0x61` | 应用场景 |
| `0x62` | 查询场景列表 |
| `0x63` | 创建 / 更新用户场景 |
| `0x64` | 删除用户场景 |
| `0x65` | 设置动态效果 |
| `0x66` | 停止动态效果 |

> 注：硬件按键 / 传感事件（如短按、长按、PIR 触发）可通过 `Type = 0x04 (EVENT)` 配合 `0x60`–`0x7F` 号段上报（见第 17.2 节示例）。

## 10.3 效果参数

效果参数采用版本化 TLV。通用字段可包括：
- EffectID；
- EffectVersion；
- Speed；
- Direction；
- Intensity；
- PaletteID；
- TransitionTime；
- 自定义参数 TLV。

未识别的可选 TLV 可以跳过；未识别的必需 TLV 必须返回 `VERSION_UNSUPPORTED` 或 `UNSUPPORTED_MODE`。

---

# 11. NOTIFY 与 EVENT 规范

## 11.1 状态 NOTIFY

NOTIFY 应报告状态变化后的实际状态。建议按属性分组：
- Power；
- Brightness；
- CCT；
- Color；
- ColorMode；
- Effect；
- Segment；
- Wi-Fi；
- OTA；
- DeviceHealth。

## 11.2 状态版本

对于需要跨 BLE / Wi-Fi 合并的状态，建议设备维护 32 位 `StateVersion`。每次业务状态变更递增。该版本可放在状态快照或扩展 TLV 中。

APP 对同一设备状态：
- 新版本覆盖旧版本；
- 相同版本视为重复；
- 旧版本不得覆盖新状态；
- 设备重启后版本处理必须定义为单调持久化或附带 BootID / Epoch；
- 若无法判断版本先后，重新 QUERY。

## 11.3 EVENT 结构

建议通用 EVENT Data（除单字节简单事件测试向量外，扩展业务事件应采用以下结构并在注册表登记）：

| 字段 | 长度 | 说明 |
|---|---:|---|
| EventID | 2B | 事件类型（Big-Endian） |
| EventVersion | 1B | 事件结构版本 |
| EventFlags | 1B | 标志位 |
| EventSequence | 4B | 设备事件序号（Big-Endian） |
| Timestamp | 4B | Unix 时间（Big-Endian）；无可信时钟时填 0 |
| Payload | N | 事件专属载荷 |

事件序号用于诊断丢失及去重，不等同于 REQUEST SEQ。

## 11.4 可靠事件

重要事件（例如 OTA 完成、配网结果、恢复出厂任务结果）不得只依赖一次性 EVENT。必须提供至少一种恢复机制：
- 任务状态 QUERY；
- 设备持久化最近任务结果；
- 事件 ACK 与重发；
- 重连后可查询的结果记录。

---

# 12. Wi-Fi 配网协议

## 12.1 适用范围

适用于 BLE + Wi-Fi 双模设备。纯 BLE 产品无需实现本章 CMD。

Wi-Fi 配网凭证必须通过安全认证后的加密会话传输。不得以明文广播或未加密 GATT Write 传输密码。

## 12.2 配网状态机

```text
IDLE
  |
  v
PROVISIONING_WINDOW
  |
  v
CREDENTIAL_RECEIVED
  |
  v
CONNECTING
  |----------------------|
  v                      v
CONNECTED               FAILED
  |                      |
  v                      v
READY                 RETRY / CANCEL
```

设备必须定义每个状态的超时、允许操作及重试策略。

## 12.3 CMD 分配

| CMD | 名称 | 功能 |
|---:|---|---|
| `0xA0` | WIFI_GET_STATUS | 查询 Wi-Fi 状态 |
| `0xA1` | WIFI_SCAN | 扫描可用网络（可选） |
| `0xA2` | WIFI_PROVISION_START | 创建配网任务 |
| `0xA3` | WIFI_SET_CREDENTIALS | 提交 SSID / 密码 |
| `0xA4` | WIFI_GET_TASK | 查询配网任务 |
| `0xA5` | WIFI_CANCEL | 取消配网任务 |
| `0xA6` | WIFI_CLEAR_CREDENTIALS | 清除网络凭证 |
| `0xA7` | WIFI_RECONNECT | 请求重新连接 |
| `0xA8` | WIFI_GET_NETWORK_INFO | 查询已连接网络摘要 |

## 12.4 凭证字段

建议使用（精确编码格式与最大长度须在 CMD 注册表冻结）：
- SSID：UTF-8 字节串，最大 32 字节（按 IEEE 802.11 标准）；
- Password：字节串，最大长度按支持的认证模式定义（如 WPA2/WPA3 最大 63/64 字节）；
- SecurityType：明确枚举；
- HiddenSSID：布尔值；
- CountryCode：按产品地区配置（如 2 字节 ISO 3166-1 alpha-2）。

密码不得回显。Response 只能返回任务接受结果和脱敏网络摘要。

## 12.5 配网错误码

业务错误可使用 Status 通用码，具体原因码枚举（须在 CMD 注册表冻结）放入 Data：
- `WIFI_NOT_FOUND`；
- `AUTHENTICATION_FAILED`；
- `DHCP_FAILED`；
- `IP_CONFLICT`；
- `UNSUPPORTED_SECURITY`；
- `INVALID_CREDENTIAL`；
- `CONNECTION_TIMEOUT`；
- `COUNTRY_NOT_SUPPORTED`。

## 12.6 配网完成

配网任务应返回 TaskID。APP 通过 EVENT 或 `WIFI_GET_TASK` 获取最终状态。设备成功连网后应更新 `STATE.WIFI_CONNECTED`，并允许通过 Wi-Fi 通道进行后续控制。

---

# 13. OTA 固件升级协议

## 13.1 OTA 原则

OTA 是独立传输子协议，不应直接把完整固件当作普通 CMD 的大载荷发送。固件包二进制封装格式（如 YBOT 信封、ECDSA-SHA256 签名与 Anti-Rollback 安全计数器）应遵循项目 [OTA 固件升级与设备管理规范.md](../6.%20工程质量与运维/OTA%20固件升级与设备管理规范.md)。

OTA 必须支持：
- 固件元信息校验；
- 设备型号 / 硬件版本匹配；
- 签名验证；
- 分块传输；
- 窗口与确认；
- 超时重传；
- 断点续传；
- 完整性校验；
- Bootloader 切换；
- 升级结果；
- 失败回滚（硬件支持时）。

## 13.2 OTA 安全要求

- 固件必须经过可信发布密钥签名。
- 设备必须在写入或启动前验证签名及目标设备范围。
- CRC 或普通哈希不能替代数字签名。
- 固件包必须包含版本、目标 PID、硬件版本范围、长度、摘要和签名信息。
- 必须防止降级到已知存在安全漏洞的版本；是否允许强制降级由授权策略决定。

## 13.3 OTA CMD 规划

| CMD | 名称 | 功能 |
|---:|---|---|
| `0xC0` | OTA_GET_STATUS | 查询 OTA 状态 |
| `0xC1` | OTA_START | 创建 OTA 任务（含固件元信息与电量前置校验） |
| `0xC2` | OTA_GET_INFO | 获取设备 OTA 接收参数（单块大小、窗口大小等） |
| `0xC3` | OTA_BEGIN | 开始数据传输 |
| `0xC4` | OTA_DATA | 发送数据块 |
| `0xC5` | OTA_QUERY_OFFSET | 查询已接收偏移（断点续传） |
| `0xC6` | OTA_END | 结束传输并触发完整性与签名验证 |
| `0xC7` | OTA_ABORT | 取消 OTA |
| `0xC8` | OTA_REBOOT | 验证通过后重启切换分区 |
| `0xC9` | OTA_RESULT | 查询升级结果 |

## 13.4 OTA 数据块

OTA_DATA 至少包含（精确字段宽度在 OTA 子协议规范/CMD 注册表冻结）：
- TransferID；
- BlockIndex 或 Offset；
- DataLength；
- Data；
- 可选块 CRC。

每次允许发送的块大小由 `OTA_GET_INFO` 返回，不得直接假设等于 ATT MTU。可使用 Write Without Response 批量发送，但必须有窗口确认和丢包恢复机制。

## 13.5 OTA 状态机

```text
IDLE
  -> PRECHECK
  -> READY
  -> TRANSFERRING
  -> VERIFYING
  -> APPLYING
  -> REBOOTING
  -> SUCCESS

任意阶段可能进入 FAILED / ABORTED
```

APP 断线后重新连接，必须查询 OTA 状态（`0xC0`）和接收偏移（`0xC5`）。设备不得仅凭 APP 本地进度条判断升级进度。

---

# 14. 产测与诊断

## 14.1 产测权限

- `0xE0`–`0xFE` 仅用于受控生产或售后诊断。
- 量产固件必须默认锁定产测指令（未解锁调用返回 `0x07 NOT_AUTHORIZED`）。
- 解锁必须使用独立授权机制，不得依赖固定公开口令。
- 产测解锁应有时效、权限和操作日志。
- 产测命令不得暴露用户凭证、私钥或安全材料。

## 14.2 诊断信息

建议提供：
- 固件版本与完整 4B Bootloader 版本；
- 硬件版本；
- 复位原因；
- BLE RSSI（由扫描端获取或诊断接口提供）；
- Wi-Fi 状态；
- 最近错误码；
- OTA 结果；
- 温度 / 电源状态（硬件支持时）。

敏感字段必须脱敏。

---

# 15. 版本兼容与扩展规则

## 15.1 协议版本

ProtocolVersion 使用 UInt16：
- 高字节：Major；
- 低字节：Minor。

示例：
- `0x0100`：V1.0；
- `0x0101`：V1.1。

由于 V1.1 帧布局改变，Major/Minor 不能单独作为帧自动识别手段；必须先通过广播版本 `AdvVer` 或独立兼容协商确认解析器。

## 15.2 兼容规则

- 未知 CMD：返回 `INVALID_COMMAND`。
- 未知可选 TLV：跳过。
- 未知必需 TLV：拒绝执行。
- 未知能力位：忽略其值，但保留原始数据。
- 未知 ModelID：使用默认 UI 资源（`0x00`）。
- 未知 PID：不猜测功能；可获取动态能力描述（`0x02` 或云端 TSL）或提示升级。
- 新固件不得改变已发布 CMD 的既有字段含义。
- 新字段必须通过版本化 TLV、CapabilityVersion 或新 CMD 扩展。
- 不得在没有版本区分的情况下改变固定长度结构。

## 15.3 CMD 注册流程

所有 CMD 必须登记在唯一协议注册表中，至少包含：
- CMD；
- 名称；
- 版本；
- 请求格式（区分 QUERY / SET 字节布局）；
- 响应格式；
- 状态码；
- 适用 PID；
- 安全等级；
- 幂等性；
- 超时策略；
- 测试向量；
- 首次发布版本。

---

# 16. APP / 固件实现要求

## 16.1 Flutter APP 模块建议

```text
ble/
  scanner/
  connection/
  gatt/
  transport/
  protocol/
    frame_codec.dart
    crc16.dart
    stream_parser.dart
    transaction_manager.dart
    command_registry.dart
  security/
  provisioning/
  ota/
  device/
    identity.dart
    capability.dart
    state.dart
  lighting/
    power.dart
    brightness.dart
    color.dart
    cct.dart
    rgbic_segment.dart
    effects.dart
```

APP 必须将 BLE 平台差异封装在 Transport 层，业务层不得直接调用平台特有 BLE API。

## 16.2 固件模块建议

```text
ble/
  advertising/
  gatt_service/
  rx_stream_parser/
  tx_scheduler/
protocol/
  frame_codec/
  crc16/
  transaction_cache/
  command_dispatcher/
security/
device/
  identity/
  capability/
  state/
lighting/
  power/
  color/
  segment/
  effect/
provisioning/
ota/
diagnostics/
```

## 16.3 日志要求

建议记录：
- 时间戳；
- DeviceID（日志中可脱敏）；
- 连接会话 ID；
- 方向；
- Type / CMD / SEQ；
- Length；
- Status；
- 超时与重试次数；
- CRC 结果；
- RSSI（若可获得）；
- 固件 / 协议版本。

不得记录：
- Wi-Fi 密码；
- 会话密钥；
- 私钥；
- 完整认证令牌；
- 未脱敏个人数据。

---

# 17. 测试向量与协议验证

## 17.1 CRC 基准测试

ASCII：

```text
123456789
```

预期 CRC：

```text
0x29B1
```

## 17.2 V1.1 帧测试向量

以下测试向量以 V1.1 帧结构生成（10 字节固定开销，2B SEQ Big-Endian）。CRC 覆盖从 `FE` 到 Data 末尾，CRC 按大端序附加。

| 场景 | 完整报文 Hex（含 CRC16） |
|---|---|
| 查询设备信息 | `FE 01 01 00 00 01 00 00 28 53` |
| 设置开灯（Operation=SET, Power=ON） | `FE 01 20 00 00 02 00 02 01 01 7A C0` |
| 设置亮度 50%（Operation=SET, Brightness=50） | `FE 01 21 00 00 03 00 02 01 32 91 72` |
| 亮度设置成功响应（实际值 50%） | `FE 02 21 00 00 03 00 01 32 98 A1` |
| 亮度状态通知（50%） | `FE 03 21 00 00 00 00 01 32 44 AE` |
| 无效 CMD 响应（CMD=0x25） | `FE 02 25 01 00 04 00 00 82 D9` |
| 长度错误响应（CMD=0x21） | `FE 02 21 02 00 05 00 00 5D 9A` |
| 按键短按事件（事件载荷示例 `01`） | `FE 04 60 00 00 00 00 01 01 E0 0B` |

上述 CRC 已按本章 CRC-16/CCITT-FALSE 参数计算。APP、固件和产测工具必须共享同一组黄金向量；新增向量应由脚本生成，并在至少两种独立实现间交叉验证。

## 17.3 必测用例

### 帧编解码
- 最小空载荷帧；
- 最大 502 Bytes 载荷；
- CRC 正确；
- CRC 错误静默丢弃；
- 非法 Head；
- 未知 Type；
- 非法 Status（含 `REQUEST` 非零 Status 与 `SEQ=0`）；
- 长度超限（`> 502`）；
- 长度与实际数据不符；
- 连续多帧解析。

### 分片与链路
- ATT MTU=23；
- ATT MTU=247；
- 协商失败；
- 一个帧跨多个 ATT Value；
- 一个 ATT Value 含多个帧；
- 分片中断；
- 半帧超时；
- 写入队列拥塞；
- Notify 订阅未完成；
- 连接中断及重连。

### 事务
- 正常 RESPONSE；
- RESPONSE 延迟；
- RESPONSE 丢失后重试；
- 重复 SEQ；
- 相同 SEQ 不同 CMD；
- 相同 SEQ 不同 Data；
- 乱序 RESPONSE；
- 未匹配 RESPONSE；
- 设备 BUSY；
- 非幂等动作超时。

### 跨平台
- iOS 扫描 / 连接 / Notify；
- Android 扫描 / 连接 / Notify；
- Android 不同厂商机型；
- iOS 前台 / 后台行为；
- BLE 地址变化；
- 不同手机重新绑定；
- 多设备同时广播；
- Scan Response 丢失；
- Wi-Fi 与 BLE 共存。

### 业务
- 开关、亮度、色温；
- RGB / RGBW / RGB+CCT；
- RGBIC 分段边界；
- 动态效果；
- 设备实体按键改变状态；
- Wi-Fi 配网成功 / 失败 / 取消；
- OTA 断点续传 / 失败 / 回滚；
- 恢复出厂及重新绑定。

---

# 18. 设备状态同步与通道仲裁

## 18.1 状态权威

设备端实际状态为最终权威状态。APP 本地状态分为：
- Desired State：用户期望状态；
- Reported State：设备上报状态；
- Pending State：等待设备确认的状态。

不得将 Desired State 直接持久化为已确认的 Reported State。

## 18.2 BLE / Wi-Fi 双通道

当 BLE 与 Wi-Fi 同时可用时：
- APP 应选择明确的主控制通道；
- 同一控制动作不得无策略地通过两个通道重复发送；
- 状态应使用设备级版本号或时间戳仲裁；
- 旧通道迟到数据不得覆盖更新状态；
- 切换通道后重新查询或同步版本；
- 云端离线控制和本地 BLE 控制必须有冲突处理规则。

## 18.3 设备主动变化

实体按键、定时任务、设备自动保护、云端控制等导致状态变化时，设备必须通过 NOTIFY 或上层 Wi-Fi 状态通道同步。重要一次性动作通过 EVENT 报告，并提供可查询结果。

---

# 19. 发布与量产验收清单

正式量产前，以下项目必须全部完成：

## 19.1 协议冻结
- [ ] V1.1 帧格式与字段长度冻结；
- [ ] 广播 `AdvVer` 与 `ProtocolVersion` 对应关系冻结；
- [ ] BID / PID / ModelID 注册表建立；
- [ ] 所有量产 CMD（含 Capability TLV、安全握手、配网、OTA 扩展载荷）已登记冻结；
- [ ] 所有枚举、单位、范围、字节序已定义；
- [ ] 测试向量自动生成并经过 C / Dart / TypeScript 交叉验证。

## 19.2 安全
- [ ] 设备唯一身份材料注入；
- [ ] 认证算法套件冻结并完成安全评审；
- [ ] 会话密钥派生与 Nonce 唯一性验证；
- [ ] 配网密码加密传输；
- [ ] 绑定 / 解绑授权；
- [ ] 防重放；
- [ ] OTA 签名验证；
- [ ] 产测接口量产锁定。

## 19.3 BLE
- [ ] iOS 与 Android 广播解析一致；
- [ ] 默认 MTU 可正常控制；
- [ ] 较大 MTU 可正常控制；
- [ ] 分片与重组测试通过；
- [ ] 多帧解析通过；
- [ ] Notify 订阅时序正确；
- [ ] BLE 地址变化不影响逻辑身份；
- [ ] 多设备并发扫描和连接测试通过。

## 19.4 业务
- [ ] 所有产品 PID 能力模型完整；
- [ ] RGB / CCT / RGBW 参数定义完整；
- [ ] RGBIC Segment 索引与边界定义完整；
- [ ] 效果 ID 与参数版本注册；
- [ ] Wi-Fi 配网状态机完成；
- [ ] OTA 状态机、断点续传及回滚完成；
- [ ] 设备状态同步和通道仲裁完成。

## 19.5 交付物
- [ ] 协议 Markdown 正式版；
- [ ] CMD 注册表（CSV / XLSX）；
- [ ] 广播解析测试数据；
- [ ] 应用帧黄金测试向量；
- [ ] C / Dart / TypeScript 编解码实现；
- [ ] 设备固件联调记录；
- [ ] iOS / Android 兼容性测试报告；
- [ ] 量产产测规范。

---

# 附录 A：关键工程决策

| 项目 | V1.1 规范 |
|---|---|
| BLE 角色 | APP Central / Device Peripheral |
| Control Service | `0xFF55` |
| Write Characteristic | `A83B0002-7D4E-4F9A-9C12-8E4B01000001` |
| Notify Characteristic | `A83B0003-7D4E-4F9A-9C12-8E4B01000001` |
| 应用帧 Head | `0xFE` |
| 帧固定开销 | 10 Bytes |
| SEQ | UInt16 Big-Endian |
| Length | UInt16 Big-Endian |
| 最大完整帧 | 512 Bytes |
| 最大 Command Data | 502 Bytes |
| CRC | CRC-16/CCITT-FALSE |
| CRC 错误 | 静默丢弃 |
| REQUEST | SEQ 非零（`0x0001`–`0xFFFF`） |
| RESPONSE | 原样回显 REQUEST SEQ |
| NOTIFY / EVENT | SEQ=0 |
| 设备身份 | DeviceID，不使用 BLE MAC 作为永久身份 |
| 设备能力 | GATT Capability Profile（CMD `0x02`） |
| 高频滑条 | WWR 中间帧 + Write 最终确认 |
| 安全 | 认证会话 + 加密完整性保护 |
| 配网 | 独立状态机与任务 ID |
| OTA | 独立传输子协议及固件签名验证 |

# 附录 B：V1.0 到 V1.1 迁移说明

| 项目 | V1.0 | V1.1 |
|---|---|---|
| AdvVer | `0x01` | `0x02` |
| ProtocolVersion | `0x0100` | `0x0101` |
| SEQ | 1 Byte | 2 Bytes |
| 帧固定开销 | 9 Bytes | 10 Bytes |
| 最大 Data | 512 Bytes | 502 Bytes（保持最大完整帧 512） |
| Query / Set | 以 Length 判定 | 每条 CMD 明确 Operation |
| CRC 错误码 | 表中出现 `0x03` | 静默丢弃，`0x03` 保留 |
| 重复请求 | 未完整定义 | 事务缓存及幂等规则 |
| 分片 | 简单流式拼包 | 明确串行化、缓存、超时与恢复 |
| 安全 | 原则性说明 | 安全握手接口与量产冻结要求 |
| 能力 | 广播 CAP | CAP + GATT Capability Profile |
| 状态同步 | NOTIFY / EVENT | 区分状态与事件，增加版本 / 任务查询 |

### 迁移强制要求

1. V1.0 与 V1.1 不得在同一连接中混用帧格式。
2. 设备广播 `AdvVer` 必须与实际协议实现一致。
3. APP 必须按 `AdvVer` 或明确的协议协商结果选择解码器。
4. 不支持 V1.1 的旧 APP 不得向 V1.1 设备发送 V1.0 控制帧。
5. 量产前必须决定是否保留 V1.0 兼容固件分支；若保留，必须明确设备端双协议识别机制和安全边界。
