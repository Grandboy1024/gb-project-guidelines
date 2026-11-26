# BLE API 通信协议规范
 
> **协议版本：** 1.0（`ProtocolVersion = 0x0101`）  

---

# 1. 总体架构与基本约定

## 1.1 字节序与编码规则

> **默认字节序声明**：除下表明确列出的小端序特殊字段（如广播 `CID` 等遵循 Bluetooth SIG 标准的字段）外，本规范所有协议帧及自定义字段中的多字节整数**默认统一使用大端序（Big-Endian）**，下文各章节不再逐一重复标注。

| 数据类型 / 字段 | 字节序 / 编码规则 | 说明 |
|---|---|---|
| **默认多字节整数** | **Big-Endian（大端序）** | 全局默认规则（适用于所有未单独声明为小端序的协议层及广播自定义字段） |
| **全部小端序字段（例外清单）** | **Little-Endian（小端序）** | 1. 主广播 `16-bit Service UUID`（Byte 5–6，2B，遵循 BLE AD 标准，如 `0xFF55` → `55 FF`）<br/>2. 主广播 `Company Identifier` / `CID`（Byte 9–10，2B，遵循 BLE AD 标准，如 `0xFFFF` → `FF FF`）<br/>*注：Bluetooth SIG 标准 GATT/ATT 控制字段（如写 `CCCD` 的 `0x0001`）在底层 ATT 报文按 SIG 标准小端序传输，通常由平台 BLE 协议栈封装处理。* |
| `MAC`、`DeviceID`、随机数、密钥、哈希摘要 | **原始字节数组** | 按字节序原样传输，不转为整数 |
| 文本字符串（SSID、名称等） | **UTF-8** | 无结尾 `\0`，最大长度按**字节数**而非字符数限制 |

## 1.2 核心通信原则

1. **应用层闭环**：需要确认的 `REQUEST` 必须收到匹配的 `RESPONSE`；GATT Write Response 仅代表 ATT 写入完成，不代表业务执行成功。
2. **设备状态权威**：设备端返回的生效状态为最终权威状态；上位机乐观更新后必须依据 `RESPONSE / NOTIFY / QUERY` 收敛。
3. **安全与分层**：`CRC16` 仅用于传输差错检测，不提供认证、加密或防重放；严禁在明文日志中打印 Wi-Fi 密码、会话密钥、私钥或完整认证令牌。
4. **能力驱动**：UI 与控制能力必须依据能力模型判断，不得仅依赖 `ModelID`、产品名称或设备名称硬编码功能。

---

# 2. BLE 广播与设备发现

## 2.1 广播类型与结构（Legacy Advertising，31 Bytes）

### BLE 广播机制

采用 BLE Legacy Advertising，主广播（Advertising Data）与扫描响应（Scan Response）各自独立限制为最大 31 Bytes，不得合并计算。

- **主广播：** 应包含设备识别及匹配所需的核心信息，APP 不得依赖设备名称或 Scan Response 完成设备识别。
- **扫描响应：** 优先用于承载设备名称（Local Name）及其他非核心信息。
- **兼容性：** 设备识别应基于主广播中定义的协议字段，Scan Response 缺失不得影响设备识别。
- **扩展广播：** 如未来启用 Extended Advertising，须单独定义协议版本及兼容策略。

### 主广播包完整逐字节布局（Byte 0 ~ Byte 30）

| 偏移 | 长度 | 字段 | 示例 / 规则 | 说明 |
|---|---:|---|---|---|
| `0` | 1B | Flags Length | `0x02` | Flags AD 段长度 |
| `1` | 1B | Flags Type | `0x01` | SIG AD Type: Flags |
| `2` | 1B | Flags Value | `0x06` | Bit 1=普通可发现，Bit 2=不支持经典蓝牙（纯 BLE） |
| `3` | 1B | UUID Length | `0x03` | 16-bit Service UUID AD 段长度 |
| `4` | 1B | UUID Type | `0x03` | SIG AD Type: Complete List of 16-bit Service UUIDs |
| `5–6` | 2B | Service UUID | `55 FF`（LE） | 控制服务 UUID `0xFF55`（小端序） |
| `7` | 1B | Manufacturer Length | `0x17` | 厂商数据段长度（23B = 1B Type + 22B Payload） |
| `8` | 1B | Manufacturer Type | `0xFF` | SIG AD Type: Manufacturer Specific Data |
| `9–10` | 2B | Company Identifier (`CID`) | `FF FF`（LE） | SIG 厂商识别码（小端序；`0xFFFF` 仅限内测，量产使用合法分配值） |
| `11` | 1B | `AdvVer` | `0x02` | 广播结构版本：`0x01`=V1.0，其他=未知 |
| `12–13` | 2B | `BID` | `0x0000` 保留 | 品牌/业务域标识 |
| `14–15` | 2B | `PID` | `0x0000` 保留 | 产品功能平台标识 |
| `16` | 1B | `ModelID` | `0x00` | 同 `(BID, PID)` 下外观款型（默认/回退 `0x00`） |
| `17` | 1B | `CAP` | Bitmap | 能力提示位图（见 2.2） |
| `18` | 1B | `STATE` | Bitmap | 即时状态位图（见 2.3） |
| `19` | 1B | `StateSeq` | `0x00–0xFF` | 状态变化模 256 序号（见 2.4） |
| `20–25` | 6B | `DeviceID_Short` | `DeviceID[2..7]` | 6 字节设备短标识（原始字节流，非密钥） |
| `26–30` | 5B | `RESERVED` | `0x00` | 预留扩展，未定义时必须填 `0x00` |
 
> **Service UUID**：Control Service 完整 UUID 为 `0000FF55-0000-1000-8000-00805F9B34FB`（仅作扫描过滤与后台发现提示，各端仍需处理系统省电与广播丢失）。

## 2.2 `CAP` 能力位图（Byte 17）

| Bit | 名称 | 定义 |
|---:|---|---|
| 0 | `BLE_CONTROL` | 支持 BLE GATT 控制 |
| 1 | `WIFI_PROVISIONING` | 支持 Wi-Fi 配网 |
| 2 | `BLE_MESH` | 支持 BLE Mesh（若实现） |
| 3 | `APP_AUTH_REQUIRED` | 控制或敏感操作需要应用层认证 |
| 4 | `OTA_SUPPORTED` | 支持通过 BLE 执行 OTA |
| 5 | `MULTI_CHANNEL` | 支持多通道灯光能力 |
| 6 | `SEGMENT_CONTROL` | 支持分段 / 像素级控制 |
| 7 | `RESERVED` | 保留，发送 0 |

## 2.3 `STATE` 状态位图（Byte 18）

| Bit | 名称 | 定义 |
|---:|---|---|
| 0 | `POWER_ON` | 主灯当前开关状态提示 |
| 1 | `BOUND` | 是否已完成用户绑定 |
| 2 | `PAIRING_WINDOW` | 是否处于允许绑定的窗口 |
| 3 | `OTA_MODE` | 是否处于 OTA / Bootloader 模式 |
| 4 | `WIFI_CONNECTED` | Wi-Fi 当前连接状态 |
| 5 | `BUSY` | 设备当前是否处于忙碌状态 |
| 6–7 | `RESERVED` | 保留，发送 0 |

> 广播 `STATE` 仅作扫描列表即时提示，建立连接后必须以 GATT 查询返回的权威状态为准。

## 2.4 `StateSeq` 模 256 规则（Byte 19）

- **固件生成**：任一 `STATE` 位变化时递增 1；普通周期重发不递增；重启后可从任意初值开始（无需持久化）。
- **比较规则**：`delta = (new_seq - old_seq) mod 256`：`0` 表示相同（过滤周期重复刷新）；`1..127` 表示新版本；`128..255` 表示可能为旧版本或跨重启/超跨度导致无法判定。
- **消费策略（防重启卡死）**：
  - 单端本地扫描可将 `128..255` 仅作为短时间窗口内的乱序防倒灌参考（超时或重新开始扫描后重置本地缓存序号），也可直接以 `new_seq != old_seq` 触发提示刷新，避免设备重启后新初值落入 `128..255` 导致状态长期不更新；
  - 存在多源广播合并（如网关转发）或强一致需求时，须严格按 `1..127` 更新、`128..255` 不直接覆盖，并在无法判定时重连 GATT 查询权威状态。

## 2.5 `DeviceID`、隐私与 Scan Response 规则

- **`DeviceID`（8 Bytes）**：出厂写入非易失存储，OTA、重启或 BLE 地址变更后保持不变；不得因解绑重新生成，**非授权密钥**。出厂/后端生成规则必须确保唯一性与高熵编码位覆盖 `DeviceID[2..7]`，严禁将主要区分位仅置于 `DeviceID[0..1]`。
- **`DeviceID_Short`（6 Bytes）**：默认取 `DeviceID[2..7]`，用于扫描列表消歧；严禁将 BLE MAC 直接用作 `DeviceID` 或使用固定单字节 XOR（如 `mac ^ 0xAD`）伪装；连接后必须通过 `CMD 0x01` 校验完整 8B `DeviceID`。若遇到 `DeviceID_Short` 相同但 `CMD 0x01` 返回的 8B `DeviceID` 不一致的设备，客户端须在本次扫描会话内按当前底层外设句柄跳过该设备并继续尝试同短 ID 候选，避免重连死循环。
- **BLE MAC 不跨平台持久化**：由于 iOS 不暴露设备 MAC 且设备可能使用可解析私有地址（RPA），绑定与重连必须以 `DeviceID` 及认证凭据为核心（注：主广播携带固定 `DeviceID_Short` 用于近场发现与消歧，不提供抗匿名广播追踪隐私）。
- **Scan Response 设备名称**：推荐格式 `SL_<ModelAbbr>_<4位短ID>`（`<4位短ID>` 固定取 `DeviceID[6..7]` 转 4 位大写十六进制字符，仅用于 UI 展示辨识，不得作唯一匹配凭据），使用 AD Type `0x09`（Complete）或 `0x08`（Shortened），UTF-8 无 `\0`，总长 ≤ 29 Bytes。
- **建议广播间隔**：未绑定/配网窗口 `100–200 ms`；已绑定可连接 `500–1000 ms`；OTA 模式须保持相同 `BID / PID / DeviceID_Short`。

---

# 3. GATT 服务与连接流程

## 3.1 GATT UUID 定义

| 项目 | UUID | 属性 |
|---|---|---|
| Control Service | `0000FF55-0000-1000-8000-00805F9B34FB` | Primary Service |
| Write Characteristic | `A83B0002-7D4E-4F9A-9C12-8E4B01000001` | Write、Write Without Response |
| Notify Characteristic | `A83B0003-7D4E-4F9A-9C12-8E4B01000001` | Notify |
| CCCD | `00002902-0000-1000-8000-00805F9B34FB` | Descriptor |

> `0xFF55` 为项目私有 16-bit Service UUID 方案，若未来需与外部生态隔离，可通过新版 128-bit Service UUID 迁移。

## 3.2 连接初始化顺序

1. 扫描广播并保留原始厂商数据；
2. 通过当前扫描句柄建立连接；
3. 发现 Control Service 与读写特征；
4. 订阅 Notify（写入 CCCD `0x0001`），**未订阅成功前不得发送依赖 Notify 的 REQUEST**；
5. 可选协商 MTU（如 247，失败不得阻断通信）；
6. 发送 `CMD 0x01` 校验完整 `DeviceID / BID / PID / ModelID`；
7. 按需执行安全认证（`CMD 0x03–0x06`）；
8. 发送 `CMD 0x02` 获取 Capability Profile；
9. 同步设备当前状态，进入 READY。

## 3.3 MTU 与单次写入长度

单次 ATT Value 上限为 `ATT_MTU - 3`，协议层不得硬编码固定 MTU，必须遵守各平台 API 返回的最大可写长度：

| ATT MTU | 单次 ATT Value 最大长度 | 单包可容纳 Command Data（`ATT_MTU - 3 - 10`） |
|---:|---:|---:|
| 23 | 20 Bytes | 10 Bytes |
| 185 | 182 Bytes | 172 Bytes |
| 247 | 244 Bytes | 234 Bytes |
| 517 | 514 Bytes | 502 Bytes |

## 3.4 写入模式、队列与断线重连

| 场景 | 写入属性 | 说明 |
|---|---|---|
| 查询、开关/亮度/色温常规设置、滑条松手终值、安全/配网/OTA 控制 | Write（With Response） | 需业务 `RESPONSE` 闭环 |
| 高频滑条拖动中间帧 | Write Without Response | 50–100 ms 节流，不重试 |
| OTA 数据块 | 按 OTA 子协议（`CMD 0xC4`） | 使用窗口确认与断点续传，不套用普通重试 |

- **队列规则**：同一连接必须使用统一串行发送调度器；同一应用帧的 ATT 分片必须连续发送，不得与其他帧交错；普通在途 `REQUEST` 建议 ≤ 4，安全/配网/OTA 必须串行。收到 `BUSY` 须退避，不得立即高频重发。
- **断线与重连**：断线时立即终止未完成事务并丢弃残片；重连后必须重新订阅 Notify、重新校验身份与认证、重新同步状态，严禁复用旧连接的 `SEQ`、会话密钥或 Nonce。

---

# 4. 应用层通用数据帧

## 4.1 帧格式（固定开销 10 Bytes，最大完整帧 512 Bytes）

| 偏移 | 字段 | 长度 | 说明 |
|---:|---|---:|---|
| `0` | `Head` | 1B | 固定 `0xFE` |
| `1` | `Type` | 1B | 消息类型（见 4.2） |
| `2` | `CMD` | 1B | 指令码 |
| `3` | `Status` | 1B | 状态码（`REQUEST` / `NOTIFY` / `EVENT` 固定 `0x00`） |
| `4–5` | `SEQ` | 2B | 事务序列号 |
| `6–7` | `Length` | 2B | `Command Data` 字节长度（`0–502`） |
| `8 .. 7+N` | `Command Data` | N B | 业务载荷（`0–502 Bytes`） |
| `8+N .. 9+N` | `CRC16` | 2B | 校验覆盖 `Head` 至 `Command Data` 末尾 |

## 4.2 Message Type 与事务匹配

| Type | 名称 | 方向 | `SEQ` / `Status` 约束 | 语义说明 |
|---:|---|---|---|---|
| `0x01` | `REQUEST` | 上位机 → 设备 | `SEQ` `0x0001–0xFFFF`；`Status=0x00` | 业务请求；不可再全局以 `Length=0` 判 Query |
| `0x02` | `RESPONSE` | 设备 → 上位机 | 回显对应 `SEQ` 与 `CMD`；`Status` 表结果 | 单 `REQUEST` 仅一个最终 `RESPONSE`；长耗时任务先返回已接受（含 `TaskID`）再靠 `EVENT`/查询跟进 |
| `0x03` | `NOTIFY` | 设备 → 上位机 | `SEQ=0x0000`；`Status=0x00` | 状态快照或状态变化推送，不承诺可靠投递 |
| `0x04` | `EVENT` | 设备 → 上位机 | `SEQ=0x0000`；`Status=0x00` | 一次性动作或异步进度事件（重要事件须支持任务查询恢复） |
| `0x05–0x7F` | `RESERVED` | — | — | 保留 |
| `0x80–0xFF` | `PRIVATE` | — | — | 未注册禁止使用 |

## 4.3 事务去重与幂等性规则

- 同一连接内未完成事务的 `SEQ` 不得复用；未匹配的 `RESPONSE` 记录诊断日志后丢弃。
- **幂等重试**：使用 Write 重试 `REQUEST` 时必须复用相同 `SEQ` 与完全相同的帧内容。设备端须维护有限事务缓存（键：`会话 + SEQ`）：
  - 同一 `SEQ` 且 `CMD` 与载荷一致：直接返回缓存 `RESPONSE`，不得重复执行；
  - 同一 `SEQ` 但 `CMD` 或载荷不同：返回 `0x0C SEQ_CONFLICT`。
- **非幂等操作**（如“切换开关”）不得在未去重时自动重试，超时后应先查询状态；业务设计应优先采用“设置目标值”。

## 4.4 CRC-16/CCITT-FALSE 算法规范

| 参数 | 定义 |
|---|---|
| Polynomial | `0x1021` |
| Initial Value | `0xFFFF` |
| RefIn / RefOut / XorOut | `False` / `False` / `0x0000` |
| 标准校验向量 | ASCII `"123456789"` → `0x29B1` |

> **CRC 错误静默丢弃**：CRC 校验失败说明报文已损坏，接收端不得返回任何 `RESPONSE`，直接丢弃并重新同步帧头。

## 4.5 流式组帧、ATT 分片与超时重试

ATT 分包不等于应用帧边界，接收端必须兼容单 ATT 多帧、单帧跨多 ATT 及半帧粘连场景：

1. **发送**：同一应用帧的 ATT 分片必须连续发送，严禁与其他帧交错；断线即丢弃未完残片。
2. **接收状态机**：找帧头 `0xFE` → 收 8B 固定头 → 校验 `Length ≤ 502`（超限立即丢弃重同步，禁止预分配超大内存） → 收齐 `10 + Length` 字节（建议半帧超时 `1000 ms`） → 校验 CRC16（错误静默丢弃） → 校验字段组合（未知 `Type` / 设备收到非 `REQUEST` / `REQUEST` 且 `SEQ==0` 直接丢弃；`REQUEST` 且 `SEQ!=0` 但 `Status!=0` 回显 `SEQ` 返回 `0x04 PARAMETER_ERROR`） → 分发并继续解析缓冲区剩余数据。
3. **超时与重试建议**：

| 操作类型 | 建议超时 | 重试策略 |
|---|---:|---|
| 普通查询 / 设置 / 滑条终值 | 1500 ms | 最多 2 次，复用原 `SEQ` |
| 安全握手 | 按阶段定义 | 不盲目重试，Nonce 严禁复用 |
| 配网提交 | 3–5 s 接受响应 | 后续查任务状态，不重复创建 |
| 高频滑条 WWR 中间帧 | 不等待 | 不重试 |

---

# 5. 状态码字典

| Status | 常量 | 含义 | 处理建议 |
|---:|---|---|---|
| `0x00` | `SUCCESS` | 成功 | 返回业务数据 |
| `0x01` | `INVALID_COMMAND` | CMD 未定义或不支持 | 拒绝执行 |
| `0x02` | `LENGTH_ERROR` | 载荷长度非法 | 拒绝执行 |
| `0x03` | `RESERVED` | 保留 | 严禁分配 |
| `0x04` | `PARAMETER_ERROR` | 参数非法或越界 | 返回错误 |
| `0x05` | `CONDITION_NOT_MET` | 当前条件不满足 | 返回错误 |
| `0x06` | `BUSY` | 设备忙或队列满 | 退避重试 |
| `0x07` | `NOT_AUTHORIZED` | 未认证或权限不足 | 拒绝执行 |
| `0x08` | `RESOURCE_ERROR` | 内存/存储资源不足 | 记录诊断 |
| `0x09` | `INTERNAL_ERROR` | 设备内部错误 | 记录诊断 |
| `0x0A` | `VERSION_UNSUPPORTED` | 协议/功能版本不支持 | 协商或提示升级 |
| `0x0B` | `OPERATION_TIMEOUT` | 设备内部操作超时 | 查询任务状态 |
| `0x0C` | `SEQ_CONFLICT` | 同 `SEQ` 对应不同请求 | 拒绝执行 |
| `0x0D` | `NOT_FOUND` | 目标对象不存在 | 检查 ID |
| `0x0E` | `ALREADY_EXISTS` | 对象已存在 | 返回已有信息 |
| `0x0F` | `RATE_LIMITED` | 请求频率超限 | 延迟后重试 |
| `0x10` | `AUTH_FAILED` | 认证失败 | 终止认证流程 |
| `0x11` | `EXPIRED` | 令牌、任务或窗口已过期 | 重新发起流程 |
| `0x12` | `UNSUPPORTED_MODE` | 当前模式不支持该操作 | 查询能力 |
| `0x13` | `INTEGRITY_ERROR` | 业务对象完整性校验失败 | 拒绝数据 |
| `0x14–0x7F` | `RESERVED` | 保留 | 注册后使用 |
| `0x80–0xFF` | `VENDOR` | 厂商私有 | 必须登记 |

---

# 6. 设备身份、基础信息与能力模型

## 6.1 `BID` / `PID` / `ModelID` 约束

- `BID`（2B）标识品牌/业务域；
- `PID`（2B）标识硬件能力与指令集边界（已发布 `PID` 不得更改协议语义）；未知 `PID` 须加载动态能力剖面，无法理解必需能力时提示升级，不得猜测控制
- `ModelID`（1B）仅区分同 `PID` 下的外观与 UI 资源；未知 `ModelID` 回退 `0x00` 默认样式


## 6.2 `CMD 0x01`：设备基础信息

- **Request**：空载荷（`Length = 0`）。
- **Response Data（固定 24 Bytes）**：

| Offset | 长度 | 字段 | 说明 |
|---|---:|---|---|
| `0–1` | 2B | `ProtocolVersion` | 协议版本 |
| `2–3` | 2B | `HardwareRevision` | 硬件版本 |
| `4–7` | 4B | `FirmwareVersion` | `Major.Minor.Patch.Build` |
| `8–15` | 8B | `DeviceID` | 全局唯一逻辑身份（原始字节） |
| `16–17` | 2B | `BID` | 品牌/业务域标识 |
| `18–19` | 2B | `PID` | 产品功能平台标识 |
| `20` | 1B | `ModelID` | 外观款型 |
| `21` | 1B | `CapabilityVersion` | 能力结构版本 |
| `22–23` | 2B | `BootloaderVersionMajorMinor` | 主次版本（完整 4B 版本及批次/SN 走诊断 CMD） |

## 6.3 `CMD 0x02`：设备能力查询

- **Request**：空载荷（`Length = 0`）。
- **Response Data**：`CapabilityVersion (1B)` + `TotalLength (2B)` + `Capability TLV List`。
- **TLV 格式**：`Tag (1B)` + `Length (2B)` + `Value (Length Bytes)`。未知可选 Tag 跳过；不支持的 CMD 必须返回 `INVALID_COMMAND` 或 `UNSUPPORTED_MODE`。

| Tag | 名称 | Value 语义（精确载荷须在 CMD 注册表冻结） |
|---:|---|---|
| `0x01` | `LIGHT_CHANNELS` | 通道数量及类型 |
| `0x02` | `COLOR_MODES` | 支持的颜色模式枚举列表 |
| `0x03` | `BRIGHTNESS_RANGE` | 最小 / 最大 / 步进 |
| `0x04` | `CCT_RANGE` | 最小 / 最大 Kelvin |
| `0x05` | `RGB_FORMAT` | RGB / RGBW / RGB+CCT 格式 |
| `0x06` | `SEGMENT_INFO` | 分段数、寻址方式、最大批量数（见 9.1） |
| `0x07` | `EFFECT_CAPABILITY` | 内置效果 ID / 参数版本 |
| `0x08` | `WIFI_CAPABILITY` | Wi-Fi 频段与配网能力 |
| `0x09` | `OTA_CAPABILITY` | 支持的 OTA 通道与块大小 |
| `0x0A` | `SECURITY_CAPABILITY` | 认证算法套件与安全版本 |
| `0x0B` | `POWER_CAPABILITY` | 电源、调光及功率限制 |

---

# 7. 安全认证、绑定与会话

## 7.1 核心安全约束

- `DeviceID`、`DeviceID_Short`、BLE MAC、设备名称均不是密钥；不得在全量固件中硬编码共享密码；不得在广播或未加密链路发送 Wi-Fi 密码、密钥或令牌。
- 认证须采用经审查的标准密码学协议，每台设备具备唯一安全身份材料（优先受保护安全存储）。
- 敏感载荷须采用标准 AEAD（如 AES-CCM / AES-GCM）保护，量产前必须在安全 profile 中冻结算法套件 ID、KDF、Nonce 格式与唯一性规则、AAD、Tag 长度及防重放计数器窗口，不得仅凭 `CAP.Bit3` 宣称已安全加固。

## 7.2 安全与绑定指令规划（`0x03`–`0x09`）

| CMD | 名称 | 职责 |
|---:|---|---|
| `0x03` | `SECURITY_CAPABILITY` | 查询安全能力与算法套件 |
| `0x04` | `AUTH_START` | 发起认证握手 |
| `0x05` | `AUTH_CHALLENGE` | 交换挑战与证明数据 |
| `0x06` | `AUTH_FINISH` | 完成双向认证并建立会话上下文（SessionID、会话密钥、防重放状态） |
| `0x07` | `SESSION_CLOSE` | 主动关闭会话 |
| `0x08` | `BIND_DEVICE` | 请求绑定（须设备处于 `BOUND=0` 且 `PAIRING_WINDOW=1` 窗口并限制窗口时长与失败次数） |
| `0x09` | `UNBIND_DEVICE` | 请求解绑（须验证当前授权，不得仅凭 `DeviceID` 执行） |

---

# 8. 通用控制指令（CMD 号段与灯光基础控制）

## 8.1 CMD 号段规划

| 号段 | 类别 | 用途 |
|---|---|---|
| `0x00` / `0xFF` | 保留 | 禁止使用 |
| `0x01–0x1F` | 系统 / 身份 / 安全 | 基础信息、能力、认证、绑定、复位、时钟 |
| `0x20–0x3F` | 基础控制 | 电源、亮度、渐变、状态 |
| `0x40–0x5F` | 颜色与光学 | CCT、颜色模式、RGB、RGBW/RGB+CCT |
| `0x60–0x7F` | 场景 / 效果 / 事件 | 场景、动态效果、硬件按键/传感事件 |
| `0x80–0x9F` | 分段 / 像素控制 | RGBIC 分段信息、分段调色、块传输 |
| `0xA0–0xBF` | Wi-Fi 配网 | 扫描、提交、任务查询、清除 |
| `0xC0–0xDF` | OTA | 固件升级 |
| `0xE0–0xFE` | 产测 / 诊断 | 工厂测试与诊断（量产默认权限锁定） |

## 8.2 核心指令字节定义

> 除显式注明外，通用控制均使用 `Operation (1B)`：`0x00 = QUERY`、`0x01 = SET`。

| CMD | 名称 | Request Data | Response / NOTIFY Data | 备注 |
|---:|---|---|---|---|
| `0x11` | 恢复出厂 | 空（`Length=0`） | `TaskID (UInt32)` | 须管理员会话；异步清理绑定与 Wi-Fi 凭证，保留 `DeviceID`，完成后断开重播；客户端须靠断线重连核验结果 |
| `0x20` | 电源控制 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `PowerState (1B: 0/1)` (2B) | `PowerState (1B: 0x00=OFF, 0x01=ON)` | — |
| `0x21` | 主亮度 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `Brightness (1B: 0–100)` (2B) | `Brightness (1B: 0–100)` | 设备限幅时返回实际生效值 |
| `0x40` | 色温 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `CCT_Kelvin (UInt16)` (3B) | `CCT_Kelvin (UInt16)` | 范围受 `CCT_RANGE` 能力约束 |
| `0x41` | 颜色模式 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `ColorMode (1B)` (2B) | `ColorMode (1B)` | `0x00=WHITE`, `0x01=CCT`, `0x02=RGB`, `0x03=RGBW`, `0x04=RGB+CCT`, `0x05=EFFECT`, `0x06=SEGMENT`；不支持返回 `0x12` |
| `0x42` | RGB 颜色 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `R(1B), G(1B), B(1B)` (4B) | `R(1B), G(1B), B(1B)` (`0–255`) | — |
| `0x43` | 扩展通道颜色 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `ChannelType(1B)` + `Count(1B)` + `Values(Count B)` | `ChannelType(1B)` + `Count(1B)` + `Values(Count B)` | 通道顺序不得在同 `PID` 内无版本变更 |
| `0x44` | 渐变参数 | QUERY: `0x00` (1B)<br/>SET: `0x01` + `TransitionTimeMs(UInt16)` + `EasingType(1B)` (4B) | `TransitionTimeMs(UInt16)` + `EasingType(1B)` | 不支持返回 `UNSUPPORTED_MODE` |

- **高频滑条闭环**：拖动过程用 `WWR`（50–100 ms 节流、不重试），松手发送带 `SEQ` 的 `Write` 终值确认；若终值失败，客户端须发起 `QUERY` 对齐设备真实状态。

---

# 9. RGBIC 与分段控制协议（`0x80`–`0x84`）

## 9.1 `SEGMENT_INFO`（Capability Tag `0x06` / `CMD 0x80`）

`CMD 0x80` Request 为空载荷（`Length=0`），Response 返回与 Capability Tag `0x06` 对齐的分段能力结构：

| 字段 | 长度 | 说明 |
|---|---:|---|
| `SegmentCount` | 2B | 逻辑分段数 |
| `AddressUnit` | 1B | `0=Segment`、`1=Pixel` |
| `MaxBatchItems` | 2B | 单条指令最大批量控制项数 |
| `IndexBase` | 1B | 起始索引（统一建议 `0`） |
| `ColorFormat` | 1B | 颜色编码格式（与 `CMD 0x81` 对齐） |
| `Flags` | 2B | 单段亮度、反向、分组等能力位 |

## 9.2 分段控制指令

| CMD | 名称 | 结构与规则 |
|---:|---|---|
| `0x81` | 分段颜色 | Request: `Operation(1B)` + `StartIndex(2B)` + `ItemCount(2B)` + `ColorFormat(1B)` + `[Values (SET 时携带)]`<br/>Response / NOTIFY: `StartIndex(2B)` + `ItemCount(2B)` + `ColorFormat(1B)` + `Values(N B)`；总载荷不得超 502B |
| `0x82` | 分段亮度 | Request: `Operation(1B)` + `StartIndex(2B)` + `ItemCount(2B)` + `[BrightnessValues (SET 时为 ItemCount B)]`<br/>Response: `StartIndex(2B)` + `ItemCount(2B)` + `BrightnessValues(ItemCount B)` |
| `0x83` | 分段效果 | 含 `SegmentRange`、`EffectID`、`EffectVersion`、`Speed`、`Direction`、`Intensity`、`EffectParameters TLV`（参数须按 `PID` 在效果注册表冻结） |
| `0x84` | 分段块传输 | 超长分段/像素数据专用应用层分块事务（含 `TransferID`、`BlockIndex`、`TotalBlocks/Size`、`Payload`、可选块校验及窗口确认/恢复机制） |

---

# 10. 场景、动态效果、NOTIFY 与 EVENT

## 10.1 场景与效果指令（`0x60`–`0x66`）

- `0x60` 查询当前场景、`0x61` 应用场景、`0x62` 查询场景列表、`0x63` 创建/更新场景、`0x64` 删除场景、`0x65` 设置动态效果、`0x66` 停止动态效果。
- 效果参数采用版本化 TLV（含 `EffectID`、`EffectVersion`、`Speed`、`Direction`、`Intensity`、`PaletteID`、`TransitionTime` 及扩展 TLV）。未识别可选 TLV 跳过，未识别必需 TLV 返回 `VERSION_UNSUPPORTED` / `UNSUPPORTED_MODE`。

## 10.2 状态版本与 `EVENT` 结构

- **状态版本号（`StateVersion`）**：需跨 BLE / Wi-Fi 合并的设备状态应维护 32 位单调递增 `StateVersion`（重启附带 `BootID / Epoch` 或持久化），新版本覆盖旧版本，无法判定时执行 `QUERY`。
- **通用 `EVENT` 载荷结构**（除单字节测试向量外，扩展事件应采用以下格式；硬件按键/传感事件归属 `0x60–0x7F` 号段）：

| 字段 | 长度 | 说明 |
|---|---:|---|
| `EventID` | 2B | 事件类型 |
| `EventVersion` | 1B | 事件结构版本 |
| `EventFlags` | 1B | 标志位 |
| `EventSequence` | 4B | 设备事件序号（用于去重与丢包诊断） |
| `Timestamp` | 4B | Unix 秒级时间戳（无时钟填 0） |
| `Payload` | N B | 事件专属载荷 |

> 重要一次性事件（配网结果、OTA 结果、恢复出厂结果）不得仅依赖不可靠 `EVENT`，设备必须保留可 `QUERY` 的任务或结果记录。

---

# 11. Wi-Fi 配网协议（`0xA0`–`0xA8`）

适用于 BLE + Wi-Fi 双模设备；**凭证必须在认证加密会话内传输**。

- **状态机**：`IDLE → PROVISIONING_WINDOW → CREDENTIAL_RECEIVED → CONNECTING → (CONNECTED → READY | FAILED → RETRY/CANCEL)`。
- **CMD 规划**：`0xA0 WIFI_GET_STATUS`、`0xA1 WIFI_SCAN`、`0xA2 WIFI_PROVISION_START`、`0xA3 WIFI_SET_CREDENTIALS`、`0xA4 WIFI_GET_TASK`、`0xA5 WIFI_CANCEL`、`0xA6 WIFI_CLEAR_CREDENTIALS`、`0xA7 WIFI_RECONNECT`、`0xA8 WIFI_GET_NETWORK_INFO`。
- **凭证约束**：`SSID` 最大 32 Bytes（UTF-8）；`Password` 按 WPA2/WPA3 规范限制（63/64 Bytes），**Response 严禁回显密码**；附带 `SecurityType`、`HiddenSSID`、`CountryCode`（2B ISO 3166-1 alpha-2）。
- **配网细化错误原因（置于 Data）**：`WIFI_NOT_FOUND`、`AUTHENTICATION_FAILED`、`DHCP_FAILED`、`IP_CONFLICT`、`UNSUPPORTED_SECURITY`、`INVALID_CREDENTIAL`、`CONNECTION_TIMEOUT`、`COUNTRY_NOT_SUPPORTED`。
- 连网成功后设备更新 `STATE.WIFI_CONNECTED` 标志，并通过 `EVENT` / `WIFI_GET_TASK` 上报任务终态。

---

# 12. OTA 固件升级与产测诊断协议

## 12.1 OTA 固件升级（`0xC0`–`0xC9`）

固件包信封格式、ECDSA-SHA256 数字签名与防回滚（Anti-Rollback）策略统一遵循 [OTA 固件升级与设备管理规范.md](../6.%20%E5%B7%A5%E7%A8%8B%E8%B4%A8%E9%87%8F%E4%B8%8E%E8%BF%90%E7%BB%B4/OTA%20%E5%9B%BA%E4%BB%B6%E5%8D%87%E7%BA%A7%E4%B8%8E%E8%AE%BE%E5%A4%87%E7%AE%A1%E7%90%86%E8%A7%84%E8%8C%83.md)，BLE 通道仅负责可靠分块传输与状态机控制：

- **状态机**：`IDLE → PRECHECK → READY → TRANSFERRING → VERIFYING → APPLYING → REBOOTING → SUCCESS`（任意阶段可进入 `FAILED / ABORTED`）。
- **CMD 规划**：`0xC0 OTA_GET_STATUS`、`0xC1 OTA_START`（含元信息与电量校验）、`0xC2 OTA_GET_INFO`（协商块大小与窗口大小）、`0xC3 OTA_BEGIN`、`0xC4 OTA_DATA`（含 `TransferID`、`BlockIndex/Offset`、`DataLength`、`Data`、可选块 CRC）、`0xC5 OTA_QUERY_OFFSET`（断点续传）、`0xC6 OTA_END`（触发签名与完整性验证）、`0xC7 OTA_ABORT`、`0xC8 OTA_REBOOT`、`0xC9 OTA_RESULT`。
- 断线重连后客户端必须通过 `0xC0` 与 `0xC5` 同步设备真实进度与偏移，严禁仅凭本地进度条判断升级结果。

## 12.2 产测与诊断（`0xE0`–`0xFE`）

- 量产固件必须默认锁定 `0xE0–0xFE`（未解锁返回 `0x07 NOT_AUTHORIZED`）；解锁必须使用有时效的独立授权机制，不得使用固定公开口令，且不得暴露用户凭证或私钥。
- 诊断信息建议覆盖：完整 4B `BootloaderVersion`、硬件版本、复位原因、RSSI、Wi-Fi 状态、最近错误码、OTA 结果及温度/电源状态（脱敏输出）。

---

# 13. 状态仲裁、版本演进与注册要求

## 13.1 状态模型与双通道仲裁

- 客户端须区分 `Desired State`（期望）、`Pending State`（等待确认）、`Reported State`（设备已确认权威状态），严禁将 `Desired State` 直接持久化为已确认状态。
- BLE 与 Wi-Fi 同时可用时，客户端必须指定唯一主控制通道（禁止无策略双通道重复下发），并基于 `StateVersion` 或时间戳仲裁，严禁旧通道迟到数据覆盖新状态；切换通道后须执行状态同步。

## 13.2 向后兼容与 CMD 注册表

- **兼容处理**：未知 CMD 返回 `0x01 INVALID_COMMAND`；未知可选 TLV 跳过；未知必需 TLV 拒绝执行；未知能力位忽略值但保留原始数据；已发布 CMD 严禁破坏性修改字段含义，扩展必须通过版本化 TLV 或新 CMD 实现。
- **CMD 注册表必填项**：CMD 编号/名称、协议版本、允许 `Message Type`、`QUERY/SET` 字节布局、成功/错误响应格式、适用 `PID`、是否需安全会话、幂等性、超时重试策略及正反测试向量。

---

# 14. 黄金测试向量与联调验证要点

## 14.1 帧测试向量（10B 固定开销）

| 场景 | 完整报文 Hex（含 CRC16） |
|---|---|
| 查询设备信息（`CMD=0x01`, `SEQ=0x0001`） | `FE 01 01 00 00 01 00 00 28 53` |
| 设置开灯（`CMD=0x20`, `SEQ=0x0002`, `SET ON`） | `FE 01 20 00 00 02 00 02 01 01 7A C0` |
| 设置亮度 50%（`CMD=0x21`, `SEQ=0x0003`, `SET 50`） | `FE 01 21 00 00 03 00 02 01 32 91 72` |
| 亮度设置成功响应（`Type=0x02`, `Brightness=50`） | `FE 02 21 00 00 03 00 01 32 98 A1` |
| 亮度状态通知（`Type=0x03`, `SEQ=0x0000`, `50`） | `FE 03 21 00 00 00 00 01 32 44 AE` |
| 无效 CMD 响应（`CMD=0x25`, `Status=0x01`） | `FE 02 25 01 00 04 00 00 82 D9` |
| 长度错误响应（`CMD=0x21`, `Status=0x02`） | `FE 02 21 02 00 05 00 00 5D 9A` |
| 按键短按事件（`Type=0x04`, `CMD=0x60`, `Payload=01`） | `FE 04 60 00 00 00 00 01 01 E0 0B` |

> 各端编解码实现必须通过上述 8 组黄金向量与 CRC 基准向量（`"123456789" → 0x29B1`）交叉验证。

## 14.2 协议联调必验边界

- **帧与分片边界**：`Length=0`、`Length=502`、`Length>502` 拒绝、CRC 错误静默丢弃、单帧跨多 ATT、单 ATT 含多帧、半帧超时重同步、断线清理未完残片。
- **事务与并发边界**：Notify 订阅完成前禁发业务帧、`SEQ` 回绕跳过 `0x0000`、相同 `SEQ` 同载荷幂等返回、相同 `SEQ` 异载荷返回 `0x0C`、`BUSY` 退避重试、高频滑条 WWR 节流 + 终值确认。
- **业务与恢复边界**：Scan Response 丢失仍可凭主广播发现设备、BLE RPA 地址变化后凭 `DeviceID` 正常重连、断线重连后 OTA `0xC0/0xC5` 断点续传、配网凭证仅走加密会话。
