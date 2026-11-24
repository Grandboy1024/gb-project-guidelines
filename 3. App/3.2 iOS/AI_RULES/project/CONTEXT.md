# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **Swift 5.9+ / Swift 6** 与 **SwiftUI** 构建的高性能原生 iOS 客户端。

应用的核心定位是面向终端用户的智能照明控制中枢与综合业务服务平台，核心设计目标包括：
1. **极致的 Apple 原生质感与交互体验**：严格遵循 Apple Human Interface Guidelines (HIG)，提供丝滑液态转场、原生态 Dynamic Type 动态字阶适配与触觉反馈（Haptics）。
2. **生产级高鲁棒性与零崩溃防御**：推行深度防御性编程与严格并发检查（Strict Concurrency Checking），对网络超时、弱网重试、长文本溢出、软键盘遮挡进行全局防护。
3. **全球化多语言完备支持**：全面基于 Apple String Catalogs (`.xcstrings`) 支持多语言环境，文案按业务 Feature 独立闭环管理。

---

## 2. 核心业务功能模块全景

所有业务功能代码统一组织在 `Features/` 目录下，各模块物理自闭环：

```
Features/
├── Auth/           # 用户认证与账户安全体系
│   ├── Login/      # 登录视图 View 与控制器 ViewModel
│   ├── Register/   # 注册视图与控制器
│   └── Data/       # Token/User 实体模型、API 请求对象与仓储
├── Home/           # 首页中枢与主导航框架
│   ├── MainTabView.swift # 基于 TabView 的三栏固定主导航
│   └── Dashboard/  # 仪表盘总览与快捷灯光控制
├── Device/         # 智能硬件设备控制与调光
│   ├── Detail/     # 设备详情、亮度/色温/RGB 调节
│   └── Settings/   # 设备固件信息与配网解绑
└── Profile/        # 个人中心与全局系统设置
    └── Settings/   # 主题模式切换（浅色/深色/跟随系统）、多语言切换与缓存清理
```

---

## 3. 基础设施与跨模块共享服务

```
Core/
├── Config/         # 环境配置加载器（基于 xcconfig 动态映射）
├── Network/        # URLSession 调度、ApiRequest 协议、统一外壳解包与拦截器
├── Storage/        # Keychain 安全持久化与 UserDefaults 封装
├── Error/          # 全局强类型密封枚举 AppException
└── Logging/        # 基于 os.Logger 的分类日志体系
```
