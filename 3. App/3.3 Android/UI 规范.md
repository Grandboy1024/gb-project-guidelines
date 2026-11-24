# FUXIN CANRUI 智能照明 App (Android 原生) — 统一设计系统规范 (design.md)

> 适用端：Android 原生客户端 (Jetpack Compose / Material 3)  
> 视觉定位：现代、极简、高阶科技感专业智能硬件风格  
> 默认外观：浅色模式（Light Mode，默认）；全流程深度适配系统级深色模式（Dark Mode）  
> 品牌名称：FUXIN CANRUI  
> 设计语言：严格遵循 Material Design 3 (M3) + 品牌专属 Design Tokens

---

## 1. 产品设计核心原则

1. **控制至上 (Control First)**：用户应当在第一时间清晰感知灯具设备状态（在线/离线、开启/关闭、当前亮度与色温）并迅速完成调控。
2. **三栏固定底导 (Simple Navigation)**：采用标准 Material 3 NavigationBar，固定三大核心底导——首页 (Home)、智能 (Smart)、我的 (Profile)。
3. **房间优先组织 (Room-First Organization)**：首页按空间/房间分组管理设备，并保留“全部设备”完整总览。
4. **全端系统一致性 (Consistent System)**：组件结构、控件交互、圆角阶梯与间距规范在所有页面严格复用。
5. **专业克制，告别过度装饰 (Professional, Not Decorative)**：采用通透平整的表面材质、精准视觉层级、轻量线性矢量图标，杜绝无意义的多重杂乱渐变与高噪点拟物光效。
6. **无障碍与易读性 (Accessible & Touch Targets)**：保障充足的色彩对比度，提供至少 48 × 48 dp 大尺寸易触控交互热区，原生支持 Edge-to-Edge 全面屏与系统无障碍字体自适应。

---

## 2. 视觉基调与规范方向

- 科技极简美学，打造专业可靠的智能硬件移动端体验。
- 默认采用通透清爽的浅色模式（Light Theme），深度支持暗室环境专用的沉稳深色模式（Dark Theme）。
- 基础底色选用冷灰蓝调净色，搭配标志性品牌科技蓝作为主交互引导色。
- 卡片采用微细边缘线（1dp border）与柔和环境微阴影，不依赖厚重视觉阴影分割内容。
- 图标库统一使用 Google 原生 **Material Icons (Outlined)**。

---

## 3. 设计色彩令牌 (Color Tokens)

在 Compose `ColorScheme` 与 `AppColors` 中严格定义：

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

全面使用 Android 原生无衬线字体 **Roboto / Google Sans**，对接 Material 3 Typography：

- **大页面标题 (Headline Large)**：`32 sp`，Bold
- **常规页面标题 (Headline Medium)**：`28 sp`，Semibold
- **分区模块标题 (Title Large)**：`20 sp`，Semibold
- **设备/卡片标题 (Title Medium)**：`16 sp`，Semibold
- **常规正文 (Body Large / Medium)**：`14 ~ 16 sp`，Regular
- **次级说明与标签 (Body Small / Label Medium)**：`12 sp`，Regular
- **数值与读数 (Tabular Metrics)**：采用等宽数字格式排版

---

## 5. 间距系统与网格 (Spacing & Grid)

基于标准的 **4dp / 8dp 基础网格**：
- `space.1`: 4 dp
- `space.2`: 8 dp
- `space.3`: 12 dp
- `space.4`: 16 dp（标准卡片内边距与横向安全间距）
- `space.5`: 20 dp
- `space.6`: 24 dp
- `space.8`: 32 dp

页面整体横向边距统一为 **16 dp**，全面屏适配必须使用 `statusBarsPadding()`、`navigationBarsPadding()` 与 `imePadding()`。

---

## 6. 圆角与表面造型 (Shape & Radius)

遵循 Material 3 几何造型规范：
- **标准设备卡片圆角**：`16 dp`
- **输入框与次级操作按钮圆角**：`12 dp`
- **全圆角胶囊 (Pill / Capsule)**：`9999 dp`，用于分类筛选标签
- **分割描边**：统一采用 `1 dp` 的微细实线边框
- **投影处理**：极柔和微阴影（1dp ~ 2dp elevation），杜绝厚重硬黑投影

---

## 7. 核心通用组件规范 (Jetpack Compose)

### 7.1 按钮 (Buttons)
- **主操作按钮 (Primary CTA)**：品牌蓝实心填充 + 白色半粗体文字，高度固定为 `48 dp`，圆角 `12 dp`。
- **次级按钮 (Secondary)**：浅色微弱表面底色搭配品牌蓝文字或 1dp 细描边边框。
- **危险操作按钮 (Destructive)**：警示红弱底配红字，仅用于解绑或删除设备。
- **触控反馈**：点击时伴随 Material 水波纹反馈与系统轻微振动反馈。

### 7.2 设备卡片 (Device Card)
必须完整展示：
- 真实设备产品图或占位矢量图
- 设备名称与空间位置标签
- 在线/离线状态指示点与文案
- 独立快捷电源开关（防抖设计）
- 快捷亮度/色温调节滑条
- 点击卡片主体通过导航控制器推入设备控制详情页

### 7.3 调光控件 (Lighting Controls)
- **亮度滑块 (Slider)**：显示实时百分比数值（如 `85%`），支持无级阻尼滑动。
- **色温滑块**：平滑暖白（2700K）至冷白（6500K）自然渐变轨道。
- **取色拾色盘**：16 种经典色预设网格 + 全光谱微调取色盘。

### 7.4 底导栏 (NavigationBar)
固定三项：
1. **首页 (Home)**：图标 `Icons.Outlined.Lightbulb`
2. **智能 (Smart)**：图标 `Icons.Outlined.AutoAwesome`
3. **我的 (Profile)**：图标 `Icons.Outlined.Person`  
高亮态使用品牌主蓝标出。

---

## 8. 设计质检清单 (Design QA Checklist)

- [ ] 默认呈现清晰舒适的浅色模式（Light Mode）。
- [ ] 每一个界面均具备严格语义对齐的深色模式（Dark Mode）。
- [ ] 底部导航严格仅包含 Home、Smart、Profile 三项。
- [ ] 所有可触控元素的响应区域均满足 48 × 48 dp 规范。
- [ ] 首页以房间分组为核心，且支持全部设备一览。
- [ ] 设备离线与开关状态无需仅依赖颜色即可清晰辨别。
- [ ] 动态长文本均已处理截断模式，适配大字体缩放。
