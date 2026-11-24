# FUXIN CANRUI 智能照明 App (iOS 原生) — 统一设计系统规范 (design.md)

> 适用端：Apple iOS 原生客户端 (SwiftUI / HIG)  
> 视觉定位：现代、极简、高阶科技感专业智能硬件风格  
> 默认外观：浅色模式（Light Mode，默认）；全流程深度适配系统级深色模式（Dark Mode）  
> 品牌名称：FUXIN CANRUI  
> 设计语言：严格遵循 Apple Human Interface Guidelines (HIG) + 品牌专属 Design Tokens

---

## 1. 产品设计核心原则

1. **控制至上 (Control First)**：用户应当在第一时间清晰感知灯具设备状态（在线/离线、开启/关闭、当前亮度与色温）并迅速完成调控。
2. **三栏固定底导 (Simple Navigation)**：采用标准原生 TabView，固定三大核心底导——首页 (Home)、智能 (Smart)、我的 (Profile)。
3. **房间优先组织 (Room-First Organization)**：首页按空间/房间分组管理设备，并保留“全部设备”完整总览。
4. **全端系统一致性 (Consistent System)**：组件结构、控件交互、圆角阶梯与间距规范在所有页面严格复用。
5. **专业克制，告别过度装饰 (Professional, Not Decorative)**：采用通透平整的表面材质、精准视觉层级、轻量线性 SF Symbols 矢量图标，杜绝无意义的多重杂乱渐变与高噪点拟物光效。
6. **无障碍与易读性 (Accessible & Dynamic Type)**：保障充足的色彩对比度，提供至少 44 × 44 pt 大尺寸易触控交互热区，原生支持 Dynamic Type 动态字体阶梯自适应。

---

## 2. 视觉基调与规范方向

- 科技极简美学，打造专业可靠的智能硬件移动端体验。
- 默认采用通透清爽的浅色模式（Light Theme），深度支持暗室环境专用的沉稳深色模式（Dark Theme）。
- 基础底色选用冷灰蓝调净色，搭配标志性品牌科技蓝作为主交互引导色。
- 卡片采用微细边缘线（1px border）与超柔超淡环境弥散投影，不依赖厚重视觉阴影分割内容。
- 图标库统一使用 Apple 原生 **SF Symbols**（统一采用 `.regular` 笔画粗细）。

---

## 3. 设计色彩令牌 (Color Tokens)

在 iOS Asset Catalog (`Assets.xcassets`) 中配置为支持 Any / Dark 外观的语义色，或在 `AppColors` 中静态调用：

### 3.1 浅色主题 (Light Theme — 默认)

| 令牌名称 (Token) | 标准色值 | 界面应用场景说明 |
|---|---:|---|
| `color.brand.primary` | `#2676D2` | 主行动点按钮 (CTA)、选中底导标签、高亮激活滑轨 |
| `color.brand.secondary` | `#25A9E0` | 辅助强调高亮、功能指示徽标 |
| `color.background` | `#F5F8FC` | 移动端应用全局浅冷灰背景底色 |
| `color.surface` | `#FFFFFF` | 内容卡片、操作面板、浮层弹窗背景 |
| `color.surface.subtle` | `#EEF3F8` | 次级浅色容器面板、文本输入框底色 |
| `color.text.primary` | `#182536` | 主标题、核心读数、主级文字 |
| `color.text.secondary` | `#718096` | 次级说明文案、未选中文本、次要提示 |
| `color.text.disabled` | `#A0AAB7` | 禁用项文本、不可选状态说明 |
| `color.border` | `#E5EBF2` | 卡片描边、功能分割线、描边输入框 |
| `color.success` | `#269B68` | 设备在线、操作成功、正常运行指示 |
| `color.warning` | `#D99128` | 警示状态、配网异常提示 |
| `color.error` | `#D94A4A` | 错误警告、危险操作（删除设备）、严重离线 |

### 3.2 深色主题 (Dark Theme)

| 令牌名称 (Token) | 标准色值 | 界面应用场景说明 |
|---|---:|---|
| `color.brand.primary` | `#4A9AFF` | 暗夜模式高对比度发光品牌蓝、激活控件 |
| `color.brand.secondary` | `#35B9F2` | 次级夜间冷蓝强调 |
| `color.background` | `#101827` | 深邃极暗背景底色 |
| `color.surface` | `#1B2638` | 深色卡片、控制板表面背景色 |
| `color.surface.subtle` | `#243247` | 深色次级面板、夜间输入框背景 |
| `color.text.primary` | `#F4F7FB` | 暗夜模式高亮清晰主文字 |
| `color.text.secondary` | `#A5B2C4` | 暗夜模式次级辅助说明文字 |
| `color.text.disabled` | `#6F7D91` | 暗夜模式置灰文字 |
| `color.border` | `#34445B` | 暗夜微细描边线与分割线 |
| `color.success` | `#48C58A` | 深色成功与在线标识 |
| `color.warning` | `#F0B34F` | 深色警告状态 |
| `color.error` | `#F07878` | 深色错误指示 |

---

## 4. 排版与文字字阶规范 (Typography)

全面使用 Apple 系统默认无衬线字体 **SF Pro**，并完整支持 Dynamic Type 动态无障碍缩放：

- **大页面标题 (Large Navigation Title)**：`34 pt`，Bold (动态跟随 `.largeTitle`)
- **常规页面标题 (Page Title)**：`28 pt`，Bold (动态跟随 `.title`)
- **分区模块标题 (Section Title)**：`20 pt`，Semibold (动态跟随 `.title3`)
- **设备/卡片标题 (Card Title)**：`16 pt`，Semibold (动态跟随 `.headline`)
- **常规正文 (Body)**：`15 pt`，Regular (动态跟随 `.body`)
- **次级说明与标签 (Supporting / Caption)**：`12 pt`，Regular (动态跟随 `.caption`)
- **数值与仪表读数 (Metrics & Numerals)**：采用系统 `.monospacedDigit()` 等宽数字排版，防数值变动引起跳动闪烁

---

## 5. 间距系统与网格 (Spacing & Grid)

基于标准的 **4pt / 8pt 基础网格**：
- `space.1`: 4 pt
- `space.2`: 8 pt
- `space.3`: 12 pt
- `space.4`: 16 pt（标准卡片内边距与横向安全间距）
- `space.5`: 20 pt
- `space.6`: 24 pt
- `space.8`: 32 pt

页面整体横向边距统一为 **16 pt**，底部必须通过安全区边距（`safeAreaInset`）适配 iPhone 底部 Home 指示条。

---

## 6. 圆角与造型体系 (Shape & Radius)

遵循 iOS 连续曲率圆角（Continuous Corner Radius）：
- **标准设备卡片圆角**：`16 pt`
- **输入框与次级操作按钮圆角**：`12 pt`
- **全圆角胶囊 (Capsule / Pill)**：用于分类筛选标签与开关按钮
- **描边粗细**：统一采用系统推荐的 `1.0 pt`（视网膜屏细线）
- **投影处理**：极柔和环境弥散阴影，严禁使用浓黑实体硬投影

---

## 7. 核心通用组件规范 (SwiftUI)

### 7.1 按钮 (Buttons)
- **主操作按钮 (Primary CTA)**：品牌蓝实心填充 + 白色半粗体文字，高度固定为 `48 pt`，圆角 `12 pt`。
- **次级按钮 (Secondary)**：浅色微弱表面底色搭配品牌蓝文字或 1pt 细描边边框。
- **危险操作按钮 (Destructive)**：警示红弱底配红字，仅用于解绑或删除设备。
- **触控反馈**：点击时必须伴随轻微系统触觉反馈（`UIImpactFeedbackGenerator(style: .light)`）。

### 7.2 设备卡片 (Device Card)
必须完整展示：
- 真实设备产品图或矢量占位图
- 设备名称与空间位置标签
- 在线/离线状态指示点与文案
- 独立快捷电源开关（内置防抖保护）
- 快捷亮度/色温调节滑条
- 点击卡片主体通过 `AppRouter` 导航推入设备详情控制页

### 7.3 调光控件 (Lighting Controls)
- **亮度滑块**：显示实时百分比数值（如 `85%`），支持无级阻尼滑动。
- **色温滑块**：平滑暖白（2700K）至冷白（6500K）渐变轨道。
- **调色轮盘**：16 种经典场景预设色块 + 原生取色拾色器。

### 7.4 底导栏 (TabView)
固定三项：
1. **首页 (Home)**：图标 `lightbulb` / `lightbulb.fill`
2. **智能 (Smart)**：图标 `wand.and.stars`
3. **我的 (Profile)**：图标 `person.crop.circle` / `person.crop.circle.fill`  
高亮态使用品牌主蓝标出。

---

## 8. 设计质检清单 (Design QA Checklist)

- [ ] 默认呈现清晰舒适的浅色模式（Light Mode）。
- [ ] 每一个界面与组件均具备语义对齐的深色模式（Dark Mode）。
- [ ] 底部导航严格仅包含 Home、Smart、Profile 三项。
- [ ] 所有可触控元素的点击感应区域均满足 44 × 44 pt 规范。
- [ ] 首页以房间分组为核心，且支持全部设备一览。
- [ ] 设备离线与开关状态无需仅依赖颜色即可清晰辨别。
- [ ] 动态长文本均已处理截断模式，适配 Dynamic Type 大字体。
