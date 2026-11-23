# Android 应用签名核心知识

---

## 1. Android 签名是什么

Android 签名是应用发布时使用数字证书和私钥生成的数字签名。系统通过签名验证应用来源及安装包完整性，并据此判断后续版本是否有资格更新已安装的应用。

签名主要用于：

1. **身份连续性**：判断新版本是否由同一签名身份发布。
2. **完整性校验**：检测签名后安装包是否被修改。
3. **应用更新验证**：防止其他签名者冒充原应用覆盖安装。

> [!NOTE]
> 签名不是应用包名，也不是应用商店账号。包名和签名共同参与应用身份及更新判断。

---

## 2. 核心术语

| 术语名称 | 含义说明 |
| :--- | :--- |
| **Keystore（密钥库）** | 保存密钥条目的文件，常见格式为 `.jks` 或 `.keystore` |
| **Private Key（私钥）** | 用于签名，必须严格保密 |
| **Certificate（证书）** | 包含公钥及证书主体等信息，用于验证签名身份 |
| **Alias（别名）** | 用于从 Keystore 中定位某个密钥条目 |
| **Store Password** | Keystore 文件访问密码 |
| **Key Password** | 某个密钥条目的密码 |
| **SHA-1 / SHA-256** | 对证书计算得到的指纹，用于识别证书 |
| **applicationId / package name** | Android 应用包名，例如 `com.example.app` |
| **versionCode** | Android 用于比较版本先后的整数版本号 |

> [!IMPORTANT]
> **重要区别：**
> - **SHA-1 / SHA-256** 是证书指纹，不是私钥，也不能用于给 APK 签名。
> - **`.jks` 文件** 可能包含私钥，属于高敏感资产。
> - **`.pem` 或 `.der` 公钥证书** 通常不包含私钥，可以用于提供证书信息。
> - 证书中的组织名称等字段本身不代表平台已验证该组织的真实性。

---

## 3. APK 签名方案

Android 支持多种 APK 签名方案：

| 方案 | 引入版本 | 说明 |
| :--- | :--- | :--- |
| **v1（JAR）** | Android 1.0 | 传统签名方案，兼容较老系统 |
| **v2** | Android 7.0 | 对 APK 整体进行完整性验证 |
| **v3** | Android 9.0 | 增加签名密钥轮换等能力 |
| **v4** | Android 11.0 | 面向增量安装等场景，通常与其他签名方案配合 |

新应用通常应根据最低支持 Android 版本和构建工具配置，启用适当的签名方案。不要仅凭“启用了 v2/v3”就假设所有渠道和设备都采用完全相同的验证路径。

> [!NOTE]
> **注意：**
> - APK 有 APK Signature Scheme。
> - AAB（Android App Bundle）是上传到 Google Play 等平台的发布格式，其上传签名与最终分发 APK 的签名角色不同。
> - AAB 上传后，Google Play 会根据 App Signing 配置生成并签署面向设备的 APK。

---

## 4. Google Play App Signing

Google Play App Signing 将密钥用途分为两类：

### 4.1 App Signing Key（应用签名密钥）

- 用于签署最终分发给用户安装的 APK。
- 决定 Google Play 分发版本的签名身份。
- 使用 Play App Signing 时，Google 负责托管并使用该密钥。
- 可以在符合 Google 当前政策和技术要求的情况下，选择提供自己的密钥。

### 4.2 Upload Key（上传密钥）

- 用于签署开发者上传到 Google Play 的 AAB 或 APK。
- Google 使用上传证书验证上传者身份。
- 它不一定是用户手机上最终 APK 的签名密钥。
- Upload Key 丢失时，通常可以按 Play Console 流程申请重置；这不等于更换 App Signing Key。

### 4.3 两种 App Signing Key 选择

#### 方式 A：让 Google 生成 App Signing Key

- **优点**：
  - Google 托管最终应用签名私钥。
  - 开发者不必自行保管该最终签名私钥。
  - 上传密钥可独立管理和重置。
- **限制**：
  - Google 分发 APK 的传统签名证书可能与开发者在其他渠道自行签名的 APK 不同。
  - 如果其他渠道使用另一把密钥，通常不能直接互相覆盖安装。

#### 方式 B：提供自己的 App Signing Key

适合需要让 Google Play 与其他渠道保持同一传统签名身份的场景。

**大致流程**：
1. 在公司受控环境生成正式应用签名密钥。
2. 按 Google Play Console 当前指引，使用官方 PEPK 工具等方式加密并提交密钥。
3. 独立设置 Upload Key。
4. 在 Play Console 核对 App Signing Certificate 的指纹。
5. 其他应用市场的最终 APK 也使用相同签名身份，并在各平台规则允许的前提下发布。

> [!IMPORTANT]
> **关键**：不要仅凭上传 AAB 使用了某个密钥，就认定 Google 分发的 APK 也使用该密钥。必须查看 Play Console 中的 **App signing certificate**，并与其他渠道最终安装包的证书指纹核对。

Google Play 的密钥升级、密钥轮换和不同 Android 版本的签名行为可能较复杂。若启用密钥升级或后量子签名等新能力，应按 Play Console 当前显示的证书和 Google 最新文档重新验证跨渠道兼容性。

---

## 5. 多应用市场统一签名策略

### 5.1 推荐目标

如果希望同一个包名的应用可以从 Google Play、华为、小米等渠道之间直接覆盖安装，建议让各渠道最终交付给用户的 APK 保持相同的签名身份。

例如：

| 渠道 | 最终 APK 签名证书 |
| :--- | :--- |
| **Google Play** | 公司指定的 App Signing Certificate |
| **华为应用市场** | 公司 Release Certificate |
| **小米应用商店** | 公司 Release Certificate |
| **OPPO / vivo** | 公司 Release Certificate |

**目标是**：上述证书指纹一致，且没有引入不兼容的签名轮换差异。

### 5.2 重要限制

- 每个应用市场对上传格式、云端签名、密钥托管和签名变更有自己的规则。
- 某个平台提供云端签名，不代表可以任意更换已发布应用的签名。
- 如果平台会重新签署上传的 APK，必须确认它使用的是哪一张证书。
- 首次上架前，先确认平台是否支持导入现有密钥，以及是否会改变最终分发证书。
- 已经上架的应用，不要在没有核查历史签名和平台政策前开启新的云端签名或更换密钥。

### 5.3 Google Play

推荐多渠道项目在首次正式发布前，评估使用自有 App Signing Key 的方案。通过 Play Console 确认是否可以提交现有密钥，并记录 Google 实际使用的 App Signing Certificate。

Upload Key 可以独立生成，不必与 App Signing Key 相同。

### 5.4 华为应用市场

AppGallery Connect 可能提供 App Signing / 云端签名能力。使用前应在当前控制台和官方文档确认：

- 是否允许使用已有签名密钥；
- 开启后最终 APK 使用哪张证书；
- 是否支持已发布应用；
- 是否存在密钥变更限制。

如果目标是跨渠道统一签名，应核对华为最终分发 APK 的证书，而不是只核对上传文件。

### 5.5 小米、OPPO、vivo 等渠道

通常需要按各平台当前上架规范提交已签名 APK，或遵循其签名托管流程。每个平台都应单独核实：

- APK 是否由平台重新签名；
- 最终分发证书是否与公司 Release Certificate 一致；
- 已发布应用是否允许更换签名；
- 更新时对包名、签名和版本号的要求。

> [!WARNING]
> 不要将未经核实的第三方教程当作平台当前政策。以应用商店开发者后台及官方文档为准。

---

## 6. 跨渠道覆盖安装的条件

例如，用户先从 Google Play 安装 V1.0，之后希望通过华为应用市场安装 V1.1。

通常需要同时满足：

1. **包名一致**：`applicationId` 相同。
2. **签名身份兼容**：签名证书一致，或 Android 系统认可其签名轮换关系。
3. **版本号递增**：新 APK 的 `versionCode` 高于已安装版本。
4. **安装包兼容**：新版本支持该设备架构、系统版本及相关功能。

如果签名不兼容，常见安装错误为：

```text
INSTALL_FAILED_UPDATE_INCOMPATIBLE
```

> [!WARNING]
> 此时通常无法直接覆盖安装。卸载旧版本再安装可能导致应用本地数据丢失，因此不要把“卸载重装”作为正常跨渠道迁移方案。

### 6.1 统一 versionCode

不同渠道应使用同一套版本号规划，避免用户从高版本渠道切换到低版本渠道时发生版本降级。

示例：

| 发布版本 | versionName | versionCode |
| :--- | :--- | :--- |
| **首发** | 1.0.0 | 100 |
| **小版本更新** | 1.1.0 | 110 |
| **功能更新** | 1.2.0 | 120 |
| **大版本更新** | 2.0.0 | 200 |

实际项目可采用其他编码规则，但必须保证每次发布递增，并确保所有渠道一致。

---

## 7. 生成正式签名密钥

使用 JDK 自带的 `keytool`：

```bash
keytool -genkeypair \
  -v \
  -keystore app-release.jks \
  -alias app-release \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000
```

参数说明：

| 参数 | 含义 |
| :--- | :--- |
| `-keystore` | 输出的密钥库文件 |
| `-alias` | 密钥条目别名 |
| `-keyalg RSA` | 密钥算法 |
| `-keysize 2048` | 密钥长度 |
| `-validity 10000` | 证书有效期（天） |

建议在生成密钥时记录并安全保存：
- Keystore 文件
- Keystore Password
- Key Password
- Alias
- 证书 SHA-256 指纹
- 生成日期、用途和责任人

> [!TIP]
> 示例文件名仅供参考。一个应用的正式密钥应长期稳定使用，不要因更换电脑或开发人员而重新生成。

---

## 8. 查看、导出和验证证书

### 8.1 查看 JKS 证书信息

```bash
keytool -list -v \
  -keystore app-release.jks \
  -alias app-release
```

重点记录 `SHA1` 和 `SHA256`。

### 8.2 导出公钥证书

```bash
keytool -exportcert \
  -rfc \
  -keystore app-release.jks \
  -alias app-release \
  -file app-release-certificate.pem
```

导出的证书不包含私钥，可以用于向平台提供证书信息。

### 8.3 验证 APK 签名

Android SDK Build Tools 提供 `apksigner`：

```bash
apksigner verify --verbose --print-certs app-release.apk
```

该命令可检查 APK 签名并输出签名证书信息。

### 8.4 比较渠道证书

对 Google Play 下载的 APK、华为下载的 APK、小米下载的 APK 分别执行：

```bash
apksigner verify --print-certs path/to/app.apk
```

> [!NOTE]
> 比较输出的证书 SHA-256。应比较**最终分发 APK**，而不是只比较上传用 AAB 或本地中间产物。

---

## 9. Flutter 项目签名配置

以下示例适用于 Flutter Android 项目。Gradle 文件可能是 Groovy（`build.gradle`）或 Kotlin DSL（`build.gradle.kts`），应根据项目实际格式配置。

### 9.1 配置 key.properties

通常放在 `android/key.properties`：

```properties
storePassword=YOUR_STORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=app-release
storeFile=/secure/path/app-release.jks
```

> [!CAUTION]
> 不要将真实密码写入代码仓库。

### 9.2 Gradle 配置示意（Kotlin DSL）

在 `android/app/build.gradle.kts` 中读取属性，并将 release build type 绑定到 release signing config。若项目已有相关配置，应合并而不是重复定义。

```kotlin
import java.util.Properties
import java.io.FileInputStream

val keystoreProperties = Properties()
val keystorePropertiesFile = rootProject.file("key.properties")

if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        create("release") {
            keyAlias = keystoreProperties.getProperty("keyAlias")
            keyPassword = keystoreProperties.getProperty("keyPassword")
            storeFile = file(keystoreProperties.getProperty("storeFile"))
            storePassword = keystoreProperties.getProperty("storePassword")
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

> [!TIP]
> 生产环境建议让 CI/CD 从安全密钥管理系统注入密钥和密码，不要把密钥文件或密码提交到仓库。

### 9.3 构建发布文件

**Google Play AAB**：

```bash
flutter build appbundle --release
```

常见输出路径：

```text
build/app/outputs/bundle/release/app-release.aab
```

**其他渠道 APK**：

```bash
flutter build apk --release
```

常见输出路径：

```text
build/app/outputs/flutter-apk/app-release.apk
```

> [!WARNING]
> 如果 Google Play 使用独立 Upload Key，而其他渠道使用 App Signing Key，应为不同发布目标配置正确的签名任务或构建配置。不要误用 Upload Key 对其他渠道 APK 签名。

---

## 10. 密钥安全管理规范

正式签名密钥是公司的核心发布资产，建议：

1. **限制访问**：仅授权发布负责人和受控 CI/CD 使用。
2. **加密备份**：至少两份加密备份，分开存放。
3. **密码分离**：密钥文件与密码不要保存在同一位置。
4. **禁止提交 Git**：严禁将 JKS、密钥密码提交到 GitHub、GitLab 等代码仓库。
5. **记录操作**：记录密钥创建、授权、使用和变更。
6. **离职交接**：人员变动时回收访问权限并检查密钥保管情况。
7. **最小权限**：开发人员不一定需要直接获得生产私钥，可通过受控签名服务完成构建。
8. **定期验证备份**：确认备份可读取、密码正确，并能在隔离环境完成签名验证。

建议在 `.gitignore` 中加入：

```gitignore
*.jks
*.keystore
key.properties
```

> [!CAUTION]
> 如果密钥或密码曾经意外提交到代码仓库，应视为可能泄露，立即评估影响并按平台密钥轮换或重置流程处理。仅删除 Git 当前版本中的文件并不能保证历史提交中的密钥已消失。

---

## 11. 常见问题

| 问题 | 说明 |
| :--- | :--- |
| **不同 APP 可以共用一把 JKS 吗？** | 技术上可以，但企业项目建议每个 APP 独立密钥，降低泄露影响范围。 |
| **同一个 APP 可以更换签名吗？** | 不能简单替换；需满足 Android 签名轮换机制和各应用市场政策。 |
| **SHA-256 能代替 JKS 吗？** | 不能。它只是证书指纹。 |
| **换电脑需要重新生成密钥吗？** | 不需要，安全迁移原密钥库及配置即可。 |
| **Upload Key 丢失怎么办？** | 通常可向 Google Play 申请重置 Upload Key。 |
| **App Signing Key 丢失怎么办？** | 取决于密钥托管方式。Play App Signing 下 Google 托管其密钥；自行管理时风险很高。 |
| **改图标、名称会影响签名吗？** | 修改应用内容后必须重新构建和签名，但不要求更换密钥。 |
| **同包名不同签名能覆盖安装吗？** | 通常不能，除非系统认可签名轮换关系。 |
| **Google Play 上传证书与分发证书为什么不同？** | Upload Key 用于验证上传者；App Signing Key 用于签署最终分发 APK。 |

---

## 12. 多渠道发布前检查清单

### 密钥与证书

- [ ] 已生成正式 Release Key。
- [ ] 已完成加密备份和恢复验证。
- [ ] 已记录 Alias、SHA-1、SHA-256。
- [ ] Google Play 的 App Signing Certificate 已核对。
- [ ] Google Upload Key 已独立配置。
- [ ] 华为、小米及其他渠道的最终分发证书已核对。

### 应用身份与版本

- [ ] 所有渠道使用相同的 `applicationId`。
- [ ] 所有渠道的 `versionCode` 统一且递增。
- [ ] 所有渠道的 `versionName` 规划一致。
- [ ] 已确认每个平台是否会重新签名。

### 安装升级测试

- [ ] Google Play 安装版本可升级到 Google Play 新版本。
- [ ] Google Play 版本可测试升级到其他渠道版本（若业务需要）。
- [ ] 华为、小米等渠道之间已进行实际覆盖安装测试。
- [ ] 已确认升级后本地数据保留。
- [ ] 已在目标 Android 版本和设备上验证。

---

## 13. 推荐的企业方案总结

对于希望同时发布到 Google Play、华为、小米、OPPO、vivo 等渠道的 Flutter APP，建议采用以下方案：

| 项目 | 推荐做法 |
| :--- | :--- |
| **每个 APP 的包名** | 固定且唯一 |
| **正式应用签名** | 公司自行生成并安全管理 |
| **Google Play App Signing** | 评估并优先采用自有 App Signing Key，以满足跨渠道签名一致需求 |
| **Google Upload Key** | 独立生成、独立保管 |
| **华为、小米等** | 按平台规则使用同一正式签名身份 |
| **版本号** | 全渠道统一管理 |
| **构建发布** | 通过受控 CI/CD 完成 |
| **验证** | 比较各渠道最终 APK 的证书指纹 |

> [!IMPORTANT]
> **最终原则：**
> 统一签名不是让所有平台上传同一个文件，而是让同一个应用在各渠道最终安装到用户设备上的 APK，具有兼容的包名、签名身份和版本关系。

---

## 14. 官方文档

- **Android 应用签名**：<https://developer.android.com/studio/publish/app-signing>
- **Android `apksigner`**：<https://developer.android.com/tools/apksigner>
- **Google Play App Signing**：<https://support.google.com/googleplay/android-developer/answer/9842756>
- **Google Play Console 帮助**：<https://support.google.com/googleplay/android-developer/>
- **华为 AppGallery Connect**：<https://developer.huawei.com/consumer/en/doc/appgallery-connect-guides/agc-appsigning-newapp-0000001052418290/>
- **小米开发者文档**：<https://dev.mi.com/>
- **Flutter Android 发布**：<https://docs.flutter.dev/deployment/android>
