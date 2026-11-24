# BLE API 通信协议规范 V1.0

> **版本：** 1.0 (正式工程基线)  
> **适用范围：** 移动端 APP（GATT Client）、智能照明设备固件（GATT Server）、产测工具  

---

## 一、基本约定

- **角色模型**：APP 为 Central / GATT Client，设备为 Peripheral / GATT Server。
- **字节序规则（严格执行）**：
  - **应用层多字节数值**（`Length`、`CRC16`、`PID`、`CCT` 等）：强制使用**大端序（Big-Endian）**。
  - **广播 CID（Company ID）**：遵循 Bluetooth SIG 标准，采用**小端序（Little-Endian）**。
  - **MAC / 字节数组**：按原始字节流顺序传输，不转为大端整数。
- **文本编码**：统一使用 UTF-8，单字段最大 32 字节，末尾**严禁**追加 `\0`。
- **核心通信原则**：
  1. **闭环应答**：每个 REQUEST 必须获得应用层 RESPONSE；GATT Write Response 仅代表传输成功，不代表业务执行成功。
  2. **状态驱动**：设备物理操作（按键、旋钮等）通过 NOTIFY / EVENT 主动上报，权威状态始终在设备端。
  3. **CRC 出错静默丢弃**：CRC 校验失败时接收端**必须直接丢弃**该帧，严禁构造错误应答。

---

## 二、BLE 广播与设备发现

广播包总长 31 字节，包含 3 字节标准 Flags 与 27 字节厂商自定义数据（AD Type: `0xFF`）。设备完整名称（Local Name）放置在 Scan Response 中。

### 2.1 厂商自定义广播结构 (AD Type: 0xFF)

| 相对偏移 | 字段名 | 长度 | 字节序 | 示例 / 取值 | 说明 |
|:---:|---|:---:|:---:|---|---|
| Byte 0~1 | `CID` | 2B | 小端序 | SIG 分配值（如 `0xFC, 0x22`） | 厂商识别码 |
| Byte 2~3 | `BID` | 2B | 大端序 | `0x0001` | 品牌/品类识别码 |
| Byte 4~5 | `PID` | 2B | 大端序 | `0x0001` ~ `0xFFFF` | 产品功能平台识别码 |
| Byte 6 | `ModelID` | 1B | - | `0x00` ~ `0xFF` | 外观/款型标识（默认基准款 `0x00`） |
| Byte 7 | `STATE` | 1B | - | Bitmap（见下表） | 广播即时状态提示位 |
| Byte 8~13 | `DeviceID_Short` | 6B | 原始流 | 6 字节逻辑标识 | 用于 APP 区分临近设备（非加密凭据） |
| Byte 14~25 | `RESERVED` | 12B | - | 全 `0x00` | 预留业务扩展 |

- **STATE 位定义**：
  - `Bit 0`：开关状态（`0b0`: 关，`0b1`: 开）
  - `Bit 1`：配网/绑定状态（`0b0`: 未绑定，`0b1`: 已绑定）
  - `Bit 2`：OTA 状态（`0b0`: 正常运行，`0b1`: OTA Bootloader 中）
  - `Bit 3~7`：保留，固定填 `0`
- **PID 与 ModelID 解耦**：`PID` 决定硬件能力集与指令集；`ModelID` 决定外观渲染与 UI 贴图。若 APP 遇到未识别的 `ModelID`，**必须自动降级加载 `0x00`（默认款）UI 资产**，严禁阻断配网。

---

## 三、GATT 服务与链路参数

### 3.1 服务与特征分配 (UUID)

| 服务 / 特征 | 属性 (Properties) | 128-bit UUID | 说明 |
|---|---|---|---|
| **Control Service** | Primary Service | `0000FF77-0000-1000-8000-00805F9B34FB` | 核心控制服务 |
| **Write Characteristic** | `Write` / `Write Without Response` | `0000FF88-0000-1000-8000-00805F9B34FB` | APP 下发请求（REQUEST） |
| **Notify Characteristic** | `Notify` | `0000FF99-0000-1000-8000-00805F9B34FB` | 设备应答 (RESPONSE) 与主动上报 |
| **CCCD** | Descriptor | `00002902-0000-1000-8000-00805F9B34FB` | 必须写入 `0x0001` 开启 Notify 监听 |

### 3.2 链路交互约束
- **有效载荷计算**：$\text{MaxDataLength} = \text{ATT\_MTU} - 3 (\text{ATT Header}) - 9 (\text{Frame Overhead})$。
  - MTU 协商为 247 时，单个 DATA 最大 **235 字节**；默认 MTU=23 时，单个 DATA 最大 **11 字节**。
- **超时与重发**：APP 请求响应超时为 **1500 ms**，超时未响应最多重试 **2 次**。重试时必须复用相同的 `Transaction ID`。
- **断线清理**：物理链路断开时，APP 必须立刻清空未完成的等待队列；重连后全量重新查询状态。

---

## 四、通用数据帧协议 (V1.0)

所有读写通信统一封装在 9 字节固定开销的协议帧内：

```text
+------+------+------+--------+-----+---------------+--------------+---------+
| Head | Type | CMD  | Status | SEQ | Length (2B)   | Command Data | CRC16   |
+------+------+------+--------+-----+---------------+--------------+---------+
| 0xFE | 1 B  | 1 B  |  1 B   | 1 B | 2 Bytes (BE)  |   N Bytes    | 2B (BE) |
+------+------+------+--------+-----+---------------+--------------+---------+
```

### 4.1 字段定义

| 字段 | 长度 | 取值范围 | 说明 |
|---|:---:|---|---|
| `Head` | 1B | 固定 `0xFE` | 帧起始标志 |
| `Type` | 1B | `0x01` ~ `0x04` | 消息类型：`0x01`=REQUEST, `0x02`=RESPONSE, `0x03`=NOTIFY, `0x04`=EVENT |
| `CMD` | 1B | `0x01` ~ `0xFE` | 业务功能指令码 |
| `Status` | 1B | `0x00` ~ `0x0C` | 状态码：REQUEST 填 `0x00`；应答填执行结果（`0x00`=SUCCESS） |
| `SEQ` | 1B | `0x00` ~ `0xFF` | 请求事务序列号：APP 循环递增分配，RESPONSE 原样回显；NOTIFY/EVENT 固定填 `0x00` |
| `Length` | 2B | `0x0000` ~ `0xFFFF` | `Command Data` 字节数 $N$（大端序） |
| `Data` | NB | 字节流 | 业务参数载荷（$N=0$ 时缺省） |
| `CRC16` | 2B | 大端序 | CRC-16/CCITT-FALSE，覆盖除 CRC 本身外的全部前 $7+N$ 字节 |

### 4.2 校验码算法 (CRC-16/CCITT-FALSE)
- **多项式**：`0x1021`，**初始值**：`0xFFFF`，**RefIn/RefOut**：False，**XorOut**：`0x0000`。
- **基准测试**：ASCII 字符串 `"123456789"` 的校验结果为 `0x29B1`。

```c
uint16_t ble_crc16_ccitt_false(const uint8_t *data, size_t len) {
    uint16_t crc = 0xFFFF;
    for (size_t i = 0; i < len; i++) {
        crc ^= (uint16_t)data[i] << 8;
        for (uint8_t b = 0; b < 8; b++) {
            crc = (crc & 0x8000) ? ((crc << 1) ^ 0x1021) : (crc << 1);
        }
    }
    return crc;
}
```

```typescript
export function crc16(buf: Uint8Array): number {
  let crc = 0xFFFF;
  for (let i = 0; i < buf.length; i++) {
    crc ^= (buf[i] << 8);
    for (let j = 0; j < 8; j++) {
      crc = (crc & 0x8000) ? ((crc << 1) ^ 0x1021) & 0xFFFF : (crc << 1) & 0xFFFF;
    }
  }
  return crc;
}
```

---

## 五、状态码字典 (Status Code)

| 状态码 | 常量名 | 含义 | 设备端行为 |
|:---:|---|---|---|
| `0x00` | `SUCCESS` | 执行成功 | 正常回显或返回查询数据 |
| `0x01` | `INVALID_COMMAND` | 指令码未定义或不支持 | 返回错误应答，Data 缺省 |
| `0x02` | `LENGTH_ERROR` | 载荷长度与该指令要求不符 | 返回错误应答，Data 缺省 |
| `0x03` | `CHECKSUM_ERROR` | CRC 错误 | **静默丢弃，禁止发送应答** |
| `0x04` | `PARAMETER_ERROR` | 参数越界或非法 | 返回错误应答，Data 缺省 |
| `0x05` | `CONDITION_NOT_MET`| 设备条件不满足（如过热受限） | 返回错误应答，Data 缺省 |
| `0x06` | `BUSY` | 设备忙，任务队列满 | 返回错误应答，APP 退避重试 |
| `0x07` | `NOT_AUTHORIZED` | 未授权 / 产测模式已锁定 | 返回错误应答，拒绝执行 |
| `0x08` ~ `0x0C` | `RESERVED` | 版本不兼容 / 资源不可用 / 内部错误等扩展 | 按预留业务错误处理 |

---

## 六、控制指令集字典

### 6.1 指令号段规划
| 号段区间 | 业务分类 | 说明 |
|---|---|---|
| `0x00` / `0xFF` | **系统保留** | 严禁分配 |
| `0x01` ~ `0x1F` | **系统管理** | 设备信息 (`0x01`)、出厂复位 (`0x11`)、时钟同步、OTA 握手 (`0x1F`) |
| `0x20` ~ `0x3F` | **基础照明** | 开关控制 (`0x20`)、主亮度调节 (`0x21`)、渐变延时 |
| `0x40` ~ `0x5F` | **色彩与光学** | 色温调节 (`0x40`)、复合混光 (`0x41`)、RGB/HSV 全彩 |
| `0x60` ~ `0x7F` | **场景与传感** | 硬件按键/传感事件 (`0x60`)、动态流光、音乐拾音 |
| `0x80` ~ `0xDF` | **业务扩展** | Mesh 编组、自动化日程 |
| `0xE0` ~ `0xFE` | **产测与调试** | 工厂校准、老化测试（量产固件必须权限锁定） |

### 6.2 常用核心指令载荷定义

#### 1. 设备基础信息 (`0x01`)
- **Query (REQUEST)**: $N=0$
- **RESPONSE 载荷**:
  - `Byte 0~1`: `ProtocolVersion` (UInt16，大端，`0x0100` 表示 V1.0)
  - `Byte 2~3`: `HardwareRevision` (UInt16，大端)
  - `Byte 4~7`: `FirmwareVersion` (UInt32，大端)
  - `Byte 8~15`: `DeviceID` (Byte[8]，唯一逻辑身份)

#### 2. 出厂恢复设置 (`0x11`)
- **Action (REQUEST)**: $N=0$
- **RESPONSE**: $N=0$（Status = `0x00` 表示确认）
- **设备行为**: 回复应答 $\to$ 清除配对凭证与用户数据（保留出厂校准参数） $\to$ 延时 200ms 断开 BLE 并恢复初始广播。

#### 3. 开关控制 (`0x20`)
- **Query (REQUEST)**: $N=0$
- **Set (REQUEST) / RESPONSE / NOTIFY 载荷 (1B)**:
  - `Byte 0`: `power_state`（UInt8，`0x00`: 关灯，`0x01`: 开灯）

#### 4. 主亮度调节 (`0x21`)
- **Query (REQUEST)**: $N=0$
- **Set (REQUEST) / RESPONSE / NOTIFY 载荷 (1B)**:
  - `Byte 0`: `brightness`（UInt8，范围 `0 ~ 100`，表示 0% ~ 100%）
  - *说明：设备应答返回实际生效亮度（考虑硬件限幅）。*

#### 5. 色温调节 (`0x40`)
- **Query (REQUEST)**: $N=0$
- **Set (REQUEST) / RESPONSE / NOTIFY 载荷 (2B)**:
  - `Byte 0~1`: `cct_kelvin`（UInt16，大端，例如 `0x0FA0` 表示 4000K）

#### 6. 复合混光参数 (`0x41`)
- **Set (REQUEST) / RESPONSE 载荷 (5B)**:
  - `Byte 0`: `mode_select`（UInt8，`0`: CCT，`1`: RGB，`2`: 流光，`3`: 自定义通道）
  - `Byte 1~4`: 各通道参数（UInt8 或 Int8 增益，由模式定义）

---

## 七、标准测试向量表 (Test Vectors)

以下 Hex 报文可直接用于跨平台（C / Dart / TypeScript）单元测试验证。

| 序号 | 业务场景 | 方向 | 报文负载 (Head+Type+CMD+Status+SEQ+Len+Data) | CRC16 | 完整报文 Hex |
|:---:|---|:---:|---|:---:|---|
| **1** | 查询设备信息 (Cmd 0x01, Seq 0x01) | APP $\to$ BT | `FE 01 01 00 01 00 00` | `34 1F` | `FE010100010000341F` |
| **2** | 设备信息应答 (Cmd 0x01, Seq 0x01, Data: 00 64) | BT $\to$ APP | `FE 02 01 00 01 00 02 00 64` | `A0 B8` | `FE0201000100020064A0B8` |
| **3** | 设置亮度 10% (Cmd 0x21, Seq 0x01, Data: 0A) | APP $\to$ BT | `FE 01 21 00 01 00 01 0A` | `CE A4` | `FE0121000100010ACEA4` |
| **4** | 亮度设置成功回显 10% (Cmd 0x21, Seq 0x01) | BT $\to$ APP | `FE 02 21 00 01 00 01 0A` | `16 26` | `FE0221000100010A1626` |
| **5** | 亮度主动上报 10% (NOTIFY 0x03, Seq 0x00) | BT $\to$ APP | `FE 03 21 00 00 00 01 0A` | `D8 F3` | `FE0321000000010AD8F3` |
| **6** | 复合混光设置 (Cmd 0x41, Seq 0x02, 5B) | APP $\to$ BT | `FE 01 41 00 02 00 05 01 02 03 04 05` | `31 C7` | `FE014100020005010203040531C7` |
| **7** | 复合混光回显 (Cmd 0x41, Seq 0x02, 5B) | BT $\to$ APP | `FE 02 41 00 02 00 05 01 02 03 04 05` | `49 3D` | `FE0241000200050102030405493D` |
| **8** | 出厂复位请求 (Cmd 0x11, Seq 0x03) | APP $\to$ BT | `FE 01 11 00 03 00 00` | `5E 25` | `FE0111000300005E25` |
| **9** | 出厂复位 ACK 应答 (Cmd 0x11, Seq 0x03) | BT $\to$ APP | `FE 02 11 00 03 00 00` | `90 C5` | `FE02110003000090C5` |
| **10** | 写入字符串 `"SmartLight"` (Cmd 0x0A, Seq 0x04) | APP $\to$ BT | `FE 01 0A 00 04 00 0A 53 6D 61 72 74 4C 69 67 68 74` | `43 9B` | `FE010A0004000A536D6172744C69676874439B` |
| **11** | 应答 - 无效指令 (Status 0x01, Seq 0x05) | BT $\to$ APP | `FE 02 25 01 05 00 00` | `D1 39` | `FE022501050000D139` |
| **12** | 应答 - 长度错误 (Status 0x02, Seq 0x06) | BT $\to$ APP | `FE 02 21 02 06 00 00` | `9A B3` | `FE0221020600009AB3` |
| **13** | 应答 - 参数错误 (Status 0x04, Seq 0x07) | BT $\to$ APP | `FE 02 21 04 07 00 00` | `8A 1A` | `FE0221040700008A1A` |
| **14** | 应答 - 条件不满足 (Status 0x05, Seq 0x08) | BT $\to$ APP | `FE 02 21 05 08 00 00` | `D0 9F` | `FE022105080000D09F` |
| **15** | 应答 - 设备忙 (Status 0x06, Seq 0x09) | BT $\to$ APP | `FE 02 21 06 09 00 00` | `7C 73` | `FE0221060900007C73` |
| **16** | 硬件按键短按上报 (EVENT 0x04, Cmd 0x60, Seq 0x00) | BT $\to$ APP | `FE 04 60 00 00 00 01 01` | `5F 6C` | `FE046000000001015F6C` |
