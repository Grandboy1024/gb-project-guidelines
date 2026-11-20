# 业务背景与工程全景 (CONTEXT)

规范编号：PROJECT-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. 产品概述与设计目标

本项目是基于 **Flutter 3.x** 与 **Dart 3.x** 构建的高性能跨平台移动端客户端，完整覆盖 **iOS** 与 **Android** 两大主流移动平台。

应用的核心定位是面向终端用户的智能硬件控制中枢与综合业务服务平台，核心设计目标包括：
1. **极致的原生质感与交互体验**：严格遵循 Material 3 设计语言，提供丝滑转场动效、触控水波纹反馈与即时手势响应。
2. **生产级高鲁棒性与零崩溃防御**：推行深度防御性编程，对网络中断、弱网重试、长文本溢出、软键盘遮挡及非法入参进行全局熔断与防护。
3. **全球化多语言完备支持**：全面支持英文、简体中文等国际化多语言环境，文案按业务 Feature 独立闭环管理。

---

## 2. 核心业务功能模块全景

所有业务功能代码统一组织在 `lib/features/` 目录下，各模块完全自闭环：

```
lib/features/
├── auth/           # 用户认证与账户安全体系
│   ├── login/      # 登录视图 View 与控制器 ViewModel
│   ├── register/   # 注册视图与控制器
│   └── data/       # Token/User 实体模型、API 请求对象与仓储
├── home/           # 首页中枢与主导航框架
│   ├── home_shell_page.dart # 基于 StatefulShellRoute 的多 Tab 状态保持导航
│   └── dashboard_page.dart  # 仪表盘总览与快捷设备控制
├── order/          # 订单中心与业务流水
│   ├── list/       # 具备下拉刷新与防并发分页的订单虚拟化列表
│   └── detail/     # 支持 DeepLink 全局唤起的强类型详情页
└── profile/        # 个人中心与全局系统设置
    └── settings/   # 主题模式切换（浅色/深色/跟随系统）、多语言切换与缓存清理
```

---

## 3. 基础设施与跨模块共享服务

```
lib/
├── app/
│   ├── app.dart           # MaterialApp.router 全局主容器与无障碍字号限幅
│   ├── router/            # GoRouter 实例装配与强类型路由表
│   ├── l10n/              # 汇总聚合各 Feature 的本地化 Delegate
│   └── theme/             # Design Tokens：ThemeData、AppSpacing、AppColors、AppRadius
├── core/                  # 与具体业务无关的纯技术基础设施
│   ├── config/env.dart    # 纯 Dart 环境集中配置（默认 prod，支持 dev 切换）
│   ├── network/           # ApiClient、ApiRequest、公共拦截器与通用响应解包
│   ├── storage/           # Key-Value 偏好存储与 Keychain 安全存储封装
│   ├── error/             # AppException 密封类异常体系
│   └── logging/           # Talker 统一日志与埋点观察者
└── shared/                # 跨 Feature 共用的无业务偏向资产
    ├── widgets/           # AppButton、AppTextField、AppDismissibleItem、AsyncValueWidget
    ├── extensions/        # Context 快捷访问扩展（l10n、colors 等）
    └── providers/         # 全局偏好 Provider（localeController、themeModeController）
```

---

## 4. 关键工程质量优先级

1. **强 MVVM 架构约束**：严禁在 Controller 中渗透 UI 元素；严禁 View 层直接跨越 Controller 访问底层仓储与网络客户端。
2. **UI 边界防护第一**：长文本必须防护黄黑溢出斑马纹；异步提交必须进行按钮状态锁定；必须闭环四态处理。
3. **自动化工具链闭环**：任何涉及数据模型、路由或文案的变更，必须通过 `build_runner` 和 `gen_l10n.dart` 生成对应的强类型代码。
