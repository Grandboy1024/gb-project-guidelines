# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **Kotlin 2.0+** 与 **Jetpack Compose (Material 3)** 构建的高性能原生 Android 客户端。

应用的核心定位是面向终端用户的智能照明控制中枢与综合业务服务平台，核心设计目标包括：
1. **极致的原生质感与交互体验**：严格遵循 Material Design 3 规范，提供平滑流体动效、Edge-to-Edge 全面屏与触觉水波纹反馈。
2. **生产级高鲁棒性与零崩溃防御**：推行深度防御性编程与结构化协程取消管控，对网络超时、弱网重试、长文本溢出、软键盘遮挡进行全局防护。
3. **全球化多语言完备支持**：全面基于 Android Resources (`strings.xml`) 支持多语言环境，文案按业务 Feature 独立闭环管理。

---

## 2. 核心业务功能模块全景

所有业务功能代码统一组织在 `features/` 目录下，各模块物理自闭环：

```
features/
├── auth/           # 用户认证与账户安全体系
│   ├── login/      # 登录视图 Screen 与控制器 ViewModel
│   ├── register/   # 注册视图与控制器
│   └── data/       # Token/User 实体模型、API 请求对象与仓储
├── home/           # 首页中枢与主导航框架
│   ├── MainScreen.kt # 基于 NavigationBar 的三栏固定主导航
│   └── dashboard/  # 仪表盘总览与快捷灯光控制
├── device/         # 智能硬件设备控制与调光
│   ├── detail/     # 设备详情、亮度/色温/RGB 调节
│   └── settings/   # 设备固件信息与配网解绑
└── profile/        # 个人中心与全局系统设置
    └── settings/   # 主题模式切换（浅色/深色/跟随系统）、多语言切换与缓存清理
```

---

## 3. 基础设施与跨模块共享服务

```
core/
├── config/         # 环境配置加载器（基于 BuildConfig 映射）
├── network/        # OkHttp 调度、ApiRequest 接口、统一外壳解包与拦截器
├── storage/        # DataStore 持久化与 KeychainStorage 加密存储
├── error/          # 全局强类型密封类 AppException
└── logging/        # 基于 Timber 的分类日志体系
```
