RESTful API 关注的是接口如何设计、资源如何组织、HTTP 方法如何使用；

REST（Representational State Transfer，表述性状态转移），是一种软件架构风格，URL 表示资源，HTTP 方法表示操作， JSON 负责传递数据。

# 一、前提

为了统一前后端的交互，提高可读性、可维护性、可扩展性;

- 遵从统一的 REST 设计风格
- 使用 JSON 作为数据交互格式
- 使用AES-256-GCM，对回调中的关键信息进行加密保护
- 所有请求必须使用 HTTPS
- 请求不应该忽略服务器证书校验的错误，避免被恶意劫持
- 有响应体时使用 application/json; charset=utf-8；204/304 无正文

## 1.1 字符集

支持 UTF-8 字符编码

## 1.2 日期格式

- 具体时刻： 统一使用 UTC，RFC 3339 格式，固定毫秒精度，例如 `2026-10-22T07:30:20.123Z`。
- 纯日期： 使用 `yyyy-MM-dd`，例如 `2026-10-22`。
- 服务端生成的时间： 如订单创建时间、更新时间、接收时间，由服务端生成。
- 时间字段由服务端生成的，客户端传入值一律忽略

### 1.2.1 用户预约当前时间

**用户预约当地时间**，固定结构如下，localTime 不带偏移，timeZone 用 IANA 名称：

```jsx
{
	"localTime":"2026-10-22T09:00:00",
	"timeZone":"America/Los_Angeles"
}
```

## 1.3 参数规范

- 命名使用：camelCase
- 路径：kebab-case
- 查询参数、JSON 字段：camelCase
- ID 一律用字符串，布尔字段用 is/has 前缀
- 空值：字段无值时返回 null，不省略；PATCH 中"缺省 = 不修改，null = 清空"
- 金额使用最小货币单位的字符串，禁止浮点
- 响应中无值字段禁止返回 null，数值类型返回 0，字符串返回空字符串，数组返回空数组

# 二、请求

## 2.1 URL  规范

1. 名词复数表达资源集合；如 `/api/v1/users` ，而不是 `~~/api/v1/user~~`
2. 多级嵌套表达 `/{父资源}/{父ID}/{子资源}/{子ID}` ；如 `GET /api/v1/users/42/orders/1001`
3. 统一使用连字符全小写（kebab-case）；如 `/api/v1/user-profiles` ，禁止使用驼峰或下划线
4. 版本控制，直接使用 `/api/v1/...`
5. 使用查询参数进行筛选、排序和分页，如： `GET /api/v1/analyses?page=2&pageSize=20&sortBy=createdAt&order=desc&status=completed`
6. **偏移分页**：page 默认 1，pageSize 默认 20、最大 100，order 取 asc|desc（默认 desc），sortBy 为各接口声明的白名单，非法值返回 400。
7. **游标分页**：大数据量或频繁变动的列表改用 cursor + limit，不返回 total。

## 2.2 HTTP 请求方法

HTTP 方法还涉及两个重要概念：

- **安全性（Safe）**：请求是否按设计不改变服务器资源状态。
- **幂等性（Idempotent）**：相同请求执行一次或多次，预期最终效果是否相同。
- **POST 幂等**：创建、支付类接口必须带 Idempotency-Key（UUID）。
    - 服务端保存 24 小时。
    - 同 key 且同请求体，返回首次结果。
    - 同 key 但请求体不同，返回 422。
- **并发控制**：GET 返回 ETag，PUT/PATCH/DELETE 带 If-Match。
    - 不匹配返回 412。
    - 缺失返回 428。

常用5种方法：

- GET：查询资源
- POST：创建资源或提交操作
- PUT：整体替换资源
- PATCH：局部修改资源
- DELETE：删除资源

| 方法 | 安全 | 幂等 |
| --- | --- | --- |
| GET | 是 | 是 |
| POST | 否 | 通常否 |
| PUT | 否 | 是 |
| PATCH | 否 | 不一定 |
| DELETE | 否 | 是 |

## 2.3 请求头部

### 2.3.1 标准请求头部

| 头部 | 必填条件 | 说明 |
| --- | --- | --- |
| `Content-Type` | 有请求体时必填 | 请求体为 JSON 时使用 `application/json`。 |
| `Accept` | 否 | 声明客户端可接受的响应媒体类型，默认 `application/json`。 |
| `Accept-Language` | 否 | 使用标准语言标签，如 `en-US`、`zh-CN`。缺失或不支持时，默认使用 `en-US`。 |
| `Authorization` | 需登录的接口必填 | 使用 Bearer 认证方案，格式为 `Bearer <accessToken>`。 |
| `Content-Digest` | 有请求体时必填 | 按 RFC 9530 计算原始请求体字节的 SHA-256 摘要，格式为 `sha-256=:<Base64>:`。 |
| `Idempotency-Key` | POST 创建、支付等需保证幂等性的接口必填 | 使用 UUID。服务端保存 24 小时。同一 Key 对应相同请求体时返回首次处理结果；对应不同请求体时返回 `422 Unprocessable Content`。 |
| `If-Match` | PUT、PATCH、DELETE 等需要并发控制的接口必填 | 使用资源 GET 响应返回的 `ETag`。不匹配时返回 `412 Precondition Failed`；要求提供但缺失时返回 `428 Precondition Required`。 |
| `User-Agent` | 否 | 用于客户端标识和问题诊断，不得作为可信身份凭证。 |

### 2.3.2 自定义请求头部

| 头部 | 必填 | 说明 |
| --- | --- | --- |
| `X-Key-Id` | 是 | 签名密钥标识，用于确定服务端验签所使用的密钥。 |
| `X-Timestamp` | 是 | 请求签名时间，使用 Unix 秒级时间戳（整数）。服务端允许的时间偏差为 ±300 秒。 |
| `X-Nonce` | 是 | 本次请求的唯一随机值，长度为 16–64 个字符，字符集为 `[A-Za-z0-9_-]`。推荐使用 128 位密码学安全随机数生成的 Base64URL 编码。相同 `X-Key-Id` 下，600 秒内不得重复。 |
| `X-Signature` | 是 | 对约定的请求签名原文进行签名，使用标准 Base64 编码。服务端通过验签检测签名覆盖内容是否被篡改，并结合时间戳和随机数防止请求重放。具体算法和签名原文见第 2.4 节。 |
| `X-Client-Version` | 是 | 客户端 APP 版本号，如 `5.2.0`。 |
| `X-OS-Version` | 是 | 手机操作系统及版本，如 `iOS/19.0`。 |
| `X-Device-Model` | 是 | 手机设备型号，如 `iPhone17`、`Xiaomi18`。 |
|  |  |  |

### 2.3.3 认证与验签规则

| 项目 | 规范 |
| --- | --- |
| Access Token | 使用 Bearer 认证方案，有效期为 2 小时。 |
| Token 刷新 | 通过 `POST /api/v1/auth/tokens/refresh` 刷新。 |
| 服务端校验顺序 | 时间戳校验 → Nonce 重放检测 → 签名验证 → Token 验证 → 权限检查。 |
| 重放防护 | 校验时间戳有效性，并确保同一 `X-Key-Id` 下的 Nonce 在规定时间窗口内不重复。 |

## 2.4 请求签名规范

### 2.4.1 签名原文

签名原文由以下 8 个字段按固定顺序组成，字段之间使用单个换行符 `\n` 连接，末尾不得添加换行符。

```
HTTP_METHOD
REQUEST_PATH
CANONICAL_QUERY
X-Timestamp
X-Nonce
X-Key-Id
CONTENT_TYPE
CONTENT_DIGEST
```

| 字段 | 规则 |
| --- | --- |
| `HTTP_METHOD` | HTTP 请求方法，统一转换为大写，如 `GET`、`POST`。 |
| `REQUEST_PATH` | 原始已编码请求路径，不包含查询字符串。 |
| `CANONICAL_QUERY` | 按第 2.4.2 节规则规范化后的查询字符串；无查询参数时为空字符串。 |
| `X-Timestamp` | 请求头 `X-Timestamp` 的原始值，使用 Unix 秒级时间戳。 |
| `X-Nonce` | 请求头 `X-Nonce` 的原始值。 |
| `X-Key-Id` | 请求头 `X-Key-Id` 的原始值。 |
| `CONTENT_TYPE` | 有请求体时使用 `Content-Type` 请求头的规范化值；无请求体时为空字符串。 |
| `CONTENT_DIGEST` | 有请求体时使用 `Content-Digest` 请求头的原始值；无请求体时固定为 `sha-256=:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=:`。 |

### 2.4.2 查询参数规范化

为确保客户端与服务端生成完全一致的签名原文，查询参数必须按照以下规则规范化：

1. **百分号编码：** 参数键和值分别按 UTF-8 编码，并遵循 RFC 3986 进行百分号编码。百分号编码中的十六进制字母必须使用大写；空格编码为 `%20`，不得使用 `+`。
2. **排序规则：** 先按参数键的编码结果升序排列；键相同时，按参数值的编码结果升序排列。比较时使用字节序。
3. **拼接规则：** 每组参数按照 `key=value` 格式拼接，各组之间使用 `&` 连接。
4. **空值处理：** 参数值为空时，仍保留等号，例如 `name=`。
5. **无参数处理：** 不存在查询参数时，`CANONICAL_QUERY` 为空字符串，不添加 `?`。
6. **重复参数：** 必须保留所有重复参数，并按照上述规则排序，不得直接转换为会丢失重复值的字典或映射结构。

示例：

```
原始查询字符串：
?page=1&name=Light

规范化结果：
name=Light&page=1
```

### 2.4.3 签名计算

使用 HMAC-SHA256 对签名原文进行消息认证，并将计算结果编码为标准 Base64。

```
SIGNATURE_BASE = 按 2.4.1 节规则生成的签名原文

X-Signature = Base64Std(
    HMAC-SHA256(SECRET_KEY, UTF8(SIGNATURE_BASE))
)
```

具体要求：

- `SECRET_KEY`：由服务端安全管理的签名密钥。
- 签名原文：使用 UTF-8 编码为字节序列。
- 签名算法：HMAC-SHA256。
- 签名编码：标准 Base64，保留必要的 `=` 填充。
- 比较方式：服务端必须使用常量时间比较方法比较计算结果与客户端提交的签名，避免因比较时间差异泄露信息。
- 安全要求：所有请求必须通过 HTTPS 传输。

### 2.4.4 服务端校验流程

服务端按照以下流程执行请求校验：

1. **时间戳校验：** 检查 `X-Timestamp` 是否为合法的 Unix 秒级整数，且与服务端时间的偏差不超过 ±300 秒。超限返回 `401 Unauthorized`，错误码为 `TIMESTAMP_EXPIRED`。
2. **签名验证：** 根据收到的请求重新生成签名原文，计算 HMAC-SHA256，并使用常量时间比较验证 `X-Signature`。不一致返回 `401 Unauthorized`，错误码为 `SIGNATURE_INVALID`。
3. **Nonce 重放检测：** 签名验证通过后，使用 Redis 原子操作登记 Nonce：
    
    ```
    SET {keyId}:{nonce} 1 NX EX 600
    ```
    
    - 操作成功：Nonce 首次使用，继续处理。
    - 操作失败且确认键已存在：返回 `401 Unauthorized`，错误码为 `NONCE_REPLAYED`。
    - Redis 不可用或操作结果不确定：应按安全策略拒绝处理或返回服务端错误，不得直接放行。
4. **Token 校验：** 按接口要求验证 Bearer Access Token 的有效性。
5. **权限检查：** 确认当前用户有权访问目标资源并执行相应操作。
6. **业务处理：** 全部校验通过后，进入业务逻辑。

**错误响应要求：** 签名验证失败时，不得在响应中泄露签名原文、密钥、摘要比较结果或具体不匹配的字段。响应应统一使用规范定义的错误结构，并包含 `X-Request-Id`。

### 2.4.5 测试向量

以下测试向量用于验证客户端与服务端的签名实现是否一致。测试密钥仅用于开发和测试，禁止用于生产环境。

| 项目 | 测试值 |
| --- | --- |
| `SECRET_KEY` | `test-secret-key` |
| HTTP 方法 | `POST` |
| 请求路径 | `/api/v1/devices` |
| 原始查询字符串 | `page=1&name=Light` |
| 规范化查询字符串 | `name=Light&page=1` |
| 请求体 | `{"deviceName":"Living Room Light"}` |
| `X-Timestamp` | `1792654220` |
| `X-Nonce` | `Z3Jhbk5vbmNlMTIzNDU2Nzg` |
| `X-Key-Id` | `k1` |
| `Content-Type` | `application/json` |
| `Content-Digest` | `sha-256=:CtXYdwJ/z+PKp8jg1eZ6yXcpbedZrxJ7xZf83d00MYI=:` |

**签名原文：**

```
POST
/api/v1/devices
name=Light&page=1
1792654220
Z3Jhbk5vbmNlMTIzNDU2Nzg
k1
application/json
sha-256=:CtXYdwJ/z+PKp8jg1eZ6yXcpbedZrxJ7xZf83d00MYI=:
```

**期望签名：**

```
WrzWJ4b0W3FNgL5jlVctTWkPIeeqCoKl27cShKbN/60=
```

验证时，客户端与服务端必须使用完全相同的请求体字节、签名原文、字符编码和 Base64 规则。任何一个字节不同，都可能导致签名不一致。

# 三、响应

## 3.1 HTTP 状态码

| 状态码 | 含义 | 典型场景 |
| --- | --- | --- |
| 200 | OK | 查询成功、修改成功 |
| 201 | Created | 资源创建成功 |
| 202 | Accepted | 请求已接受，任务尚未完成 |
| 204 | No Content | 操作成功，没有响应正文 |
| 400 | Bad Request | 请求参数或请求格式不符合要求 |
| 401 | Unauthorized | 缺少有效身份认证 |
| 403 | Forbidden | 已识别身份，但无权执行操作 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源状态冲突或出现重复创建冲突 |
| 422 | Unprocessable Content | 请求内容无法通过语义或数据校验 |
| 429 | Too Many Requests | 请求过于频繁 |
| 500 | Internal Server Error | 服务器内部错误 |
| 503 | Service Unavailable | 服务暂时不可用 |

## 3.2 响应体

### 3.2.1 成功

常规：

```jsx
{
  "id": "device-001",
  "deviceName": "Living Room Light",
  "isOnline": true
}
```

分页列表：

```jsx
{
  "items": [
    {
      "id": "device-001",
      "deviceName": "Living Room Light",
      "isOnline": true
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 100
}
```

### 3.2.2 失败 4xx 或 5xx

- * code: 业务对应的错误码
- * message: 会根据语言返回对应的错误信息，可以直接展示出来
- details: 具体错误的原因

```jsx
{
  "code": "DEVICE_NOT_FOUND",
  "message": "The requested device was not found.",
  "details": [
    {
      "target": "deviceId",
      "reason": "RESOURCE_NOT_FOUND"
    }
  ]
}
```

## 3.3 回调加密

### 3.3.1 算法参数

| 项目 | 规则 |
| --- | --- |
| 加密算法 | AES-256-GCM。 |
| 加密密钥 | 32 字节，按 `keyId` 标识和管理，与请求签名密钥分开存储、使用和轮换。 |
| IV（初始化向量） | 固定 12 字节，每次加密使用密码学安全随机数生成器（CSPRNG）生成。同一密钥下不得重复使用 IV。 |
| Authentication Tag | 固定 16 字节，用于验证密文的完整性和真实性。 |
| AAD（附加认证数据） | 将 `keyId` 与 `eventId` 以 `.` 连接，再使用 UTF-8 编码。例如：`k1.evt-20261022-0001`。AAD 不加密，但必须参与认证。 |
| 明文 | 业务 JSON 按 UTF-8 编码得到的字节序列。 |
| 密文 | 使用 AES-256-GCM 对明文字节加密后得到的密文字节，不包含 IV 和 Authentication Tag。 |

### 3.3.2 加密传输格式

加密结果按照以下格式拼接，并使用标准 Base64 编码：

```
encrypted = Base64Std(IV || ciphertext || tag)
```

| 字段 | 长度或格式 | 说明 |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
| `IV` | 固定 12 字节 | 初始化向量。 |  |  |  |  |
| `ciphertext` | 可变长度 | AES-256-GCM 生成的密文。 |  |  |  |  |
| `tag` | 固定 16 字节 | 认证标签。 |  |  |  |  |
| `encrypted` | 标准 Base64 | 对 `IV |  | ciphertext |  | tag`的完整字节序列进行 Base64 编码，保留必要的`=` 填充。 |

接收方解码 `encrypted` 后，必须按照上述固定顺序提取 IV、密文和认证标签。不得将 IV 或认证标签省略，也不得调整拼接顺序。

### 3.3.3 回调请求体

回调请求体采用 JSON 格式，结构如下：

```
{
  "keyId": "k1",
  "eventId": "evt-20261022-0001",
  "eventTime": "2026-10-22T07:30:20.123Z",
  "encrypted": "<Base64>"
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `keyId` | String | 是 | 加密密钥标识，用于选择对应的 AES-256-GCM 密钥。 |
| `eventId` | String | 是 | 业务事件的唯一标识，用于事件去重和处理状态管理。 |
| `eventTime` | String | 是 | 事件发生时间，使用 UTC 时区的 RFC 3339 格式，固定精确到毫秒，以 `Z` 结尾。 |
| `encrypted` | String | 是 | 按第 3.3.2 节格式生成的加密数据。 |

### 3.3.4 接收方处理流程

1. **参数校验：** 检查回调请求体结构、必填字段、字段格式及数据长度。校验 `keyId` 是否有效，并根据 `keyId` 获取对应的解密密钥。
2. **解码与解密：** 对 `encrypted` 执行标准 Base64 解码，按固定格式提取 IV、密文和认证标签；使用 `keyId + "." + eventId` 的 UTF-8 字节作为 AAD，执行 AES-256-GCM 解密。
3. **认证校验：** 如果认证标签校验失败，必须丢弃该请求，不得使用或处理解密结果，并返回 `400 Bad Request`。不得在认证成功前将解密结果交给业务逻辑。
4. **事件去重：** 解密和认证成功后，按事件唯一标识 `eventId` 检查处理记录。已成功处理的重复事件直接返回 `200 OK`，不得重复执行业务操作。
5. **业务处理：** 对尚未成功处理的事件执行业务逻辑，并持久化处理结果。应确保事件去重与业务处理具备必要的并发控制和事务保障。
6. **返回结果：** 处理成功返回 `200 OK` 或 `204 No Content`。发生可重试的临时错误时返回约定的非成功状态码，由发送方按重试策略重新投递。

### 3.3.5 回调重试与幂等

| 项目 | 规则 |
| --- | --- |
| 成功响应 | `200 OK` 或 `204 No Content` 表示回调处理成功。 |
| 重复事件 | 已成功处理的 `eventId` 再次到达时，直接返回 `200 OK`。 |
| 失败重试 | 发送方应对约定的可重试失败进行重试，并采用有上限的退避策略。 |
| 幂等处理 | 接收方应以 `eventId` 为幂等键，避免回调重试导致业务重复执行。 |
| 处理状态 | 建议持久化事件处理状态，区分处理中、处理成功和可重试失败，避免并发重复处理。 |

### 3.3.6 密钥轮换

| 场景 | 规则 |
| --- | --- |
| 常规轮换 | 新旧密钥并行有效至少 7 天。发送方切换到新 `keyId` 后，接收方仍须能够使用旧密钥解密轮换期间到达的合法回调。 |
| 密钥选择 | 接收方根据回调中的 `keyId` 选择对应密钥，不得尝试使用任意其他密钥解密。 |
| 旧密钥退役 | 并行期结束且确认不再需要处理旧密钥对应的合法回调后，方可停用旧密钥。 |
| 疑似泄露 | 立即停用并作废受影响的密钥，不适用常规并行期；必要时重新生成密钥、更新配置并排查异常请求。 |
| 密钥安全 | 密钥必须安全生成、存储和分发，不得明文写入日志、代码仓库或回调请求体。 |