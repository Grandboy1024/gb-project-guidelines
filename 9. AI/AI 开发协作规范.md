核心目标是：
1. 一套规范，适用于所有 AI 工具。
2. 一份主规范，避免不同平台的指令逐渐产生差异。
3. 不同 AI 工具只负责读取和适配规范，不自行重新定义规范。
4. 项目技术决策、开发进度、代码规范能够在不同 AI 之间延续。
5. 更换模型时，不需要重新向 AI 解释整个项目。

采用 通用规范 + 平台适配层 + 项目知识库 三层架构

# 一、整体架构：一份主规范，多个入口文件

使用目录结构：
```
my-project/
│
├── AI_RULES/
│   ├── RULES.md                 # 规范总入口 / 规范清单
│   │
│   ├── core/
│   │   ├── MASTER.md            # 核心行为规范
│   │   ├── WORKFLOW.md          # 工作流程
│   │   └── COMMUNICATION.md     # 沟通规范
│   │
│   ├── development/
│   │   ├── CODING.md            # 编码规范
│   │   ├── ARCHITECTURE.md      # 架构规范
│   │   └── TESTING.md           # 测试规范
│   │
│   ├── project/
│   │   ├── CONTEXT.md           # 项目背景
│   │   ├── DECISIONS.md         # 技术决策
│   │   └── STATUS.md            # 项目状态
│   │
│   └── CHANGELOG.md             # 规范变更记录
│
├── AGENTS.md                   # Codex 入口
├── CLAUDE.md                   # Claude 入口
├── GEMINI.md                   # Gemini 入口
│
└── src/
```

# 二、AI 文件约束

## 2.1 RULES.md：统一规范清单

```markdown
# AI 规范清单 (Rules Manifest)

版本：1.0.0

## 1. 目的

本文档是所有 AI 规则和项目开发指令的唯一主入口。

所有 AI 工具在执行项目任务前，必须首先阅读并遵循本清单。

本清单明确了各项文档的权威性以及适用的规范边界。

## 2. 强制性核心规范

以下文档对所有 AI 工具均为强制执行：

| 编号 | 文件路径 | 规范描述 |
|---|---|---|
| CORE-001 | core/MASTER.md | AI 核心行为准则与红线禁令 |
| CORE-002 | core/WORKFLOW.md | 标准作业流程 (SOP) |
| CORE-003 | core/COMMUNICATION.md | 沟通交互与交付规范 |

## 3. 开发规范

在执行软件开发、代码编写、重构或测试任务时，以下文档为强制性规范：

| 编号 | 文件路径 | 规范描述 |
|---|---|---|
| DEV-001 | development/CODING.md | 编码与防御规范 |
| DEV-002 | development/ARCHITECTURE.md | 架构与分层规范 |
| DEV-003 | development/TESTING.md | 测试与质量保证规范 |

## 4. 项目知识库

以下文档提供项目特定的业务背景与历史决策：

| 编号 | 文件路径 | 规范描述 |
|---|---|---|
| PROJECT-001 | project/CONTEXT.md | 项目背景与业务全景 |
| PROJECT-002 | project/DECISIONS.md | 已确认的技术架构决策 (ADR) |
| PROJECT-003 | project/STATUS.md | 项目当前进展与状态 |

## 5. 执行要求

1. 在开始任何任务前，务必先阅读本清单。
2. 阅读与当前任务相关的所有强制性文档。
3. 严禁忽略任何适用的规范。
4. 严禁在未阅读文档的情况下盲目假设已知晓内容。
5. 若引用的文档不存在，必须立即向开发者报告缺失文件。
6. 未经开发者明确授权，严禁擅自修改规范。

## 6. 冲突优先级

若不同指令或文档之间发生冲突，严格按以下优先级处理：

1. 系统与平台安全、运行环境边界要求。
2. 开发者针对当前任务的明确显式指令。
3. 已确认的项目需求与架构决策 (`project/DECISIONS.md`)。
4. 核心行为准则 (`core/MASTER.md`)。
5. 任务相关的开发规范 (`development/`)。
6. 项目知识与背景文档 (`project/`)。

## 7. 维护要求

本文档为项目规范的权威登记册。
当新增、删除或重命名任何规范文档时，必须同步更新本清单并记录变更。
```

## 2.2 GEMINI.md
```markdown
# AI 指令入口

@AI_RULES/RULES.md

在执行任何项目任务前，请遵循上述清单并阅读所有适用的规范文档。
```

## 2.3 AGENTS.md
```markdown
# AI 指令入口

在执行任何项目任务前：

1. 阅读 `AI_RULES/RULES.md`。
2. 遵循清单确定当前任务适用的具体规范。
3. 完整阅读所有强制性及任务相关的文档。
4. 严格遵循最新的适用规则进行开发。

禁止自行重新定义或违背项目既定规则。
```

## 2.4 CLAUDE.md
```markdown
# AI 指令入口

@AI_RULES/RULES.md

在执行任何项目任务前，请遵循上述清单并阅读所有适用的规范文档。
```

# 三、Flutter APP 落地实践

在 Flutter App 项目中，完整实例化上述三层 AI 协作架构。详细的 Flutter 开发与架构规范请查阅 [Flutter 项目规范.md](../3.%20App/Flutter%20项目规范.md)，AI 规范落地对应 `3. App/AI_RULES/` 目录。

## 3.1 Flutter 工程 AI 规范文件映射

| 规范分层 | 对应文件路径 | 核心内容与标准 |
| --- | --- | --- |
| **总清单 (Manifest)** | `AI_RULES/RULES.md` | 统领所有规则，声明必须遵循的技术选型与执行优先级 |
| **核心层 (Core)** | `AI_RULES/core/MASTER.md` | AI 行为准则与十条绝对禁止的硬性红线 |
| | `AI_RULES/core/WORKFLOW.md` | 5 阶段标准化研发流程（认知 -> 方案 -> 编码 -> 工具链 -> 质量门禁） |
| | `AI_RULES/core/COMMUNICATION.md` | 交互规范、代码变更呈现与决策确认标准 |
| **开发层 (Development)**| `AI_RULES/development/CODING.md` | Dart 3 语法、UI Design Tokens、输入框/列表/防抖/边界防御 |
| | `AI_RULES/development/ARCHITECTURE.md` | Clean MVVM、Riverpod 2.x 代码生成、Typed GoRouter、Dio 请求对象 |
| | `AI_RULES/development/TESTING.md` | ProviderContainer 测试、Repository Mock、Widget 测试与质量校验 |
| **项目知识层 (Project)** | `AI_RULES/project/CONTEXT.md` | 产品定位、模块全景与基础设施概述 |
| | `AI_RULES/project/DECISIONS.md` | 架构决策记录（ADR-001 ~ ADR-008） |
| | `AI_RULES/project/STATUS.md` | 当前阶段、已落地模块与技术债务管理 |
| **变更记录** | `AI_RULES/CHANGELOG.md` | 规范变更与版本演进记录 |
| **平台适配入口** | `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` | 分别适配 Codex/Copilot、Claude Code、Gemini/Antigravity |

## 3.2 核心执行原则

1. **唯一真相源**：所有 AI 针对 Flutter 项目的代码修改，必须以 `AI_RULES/RULES.md` 为权威清单，禁止脱离规范盲目编写。
2. **自动化工具链联动**：每次涉及模型或路由修改必须执行 `dart run build_runner build -d`，涉及多语言文案必须执行 `dart run tool/gen_l10n.dart`。
3. **质量闭环**：代码交付前必须确保 `dart format`、`flutter analyze` 与 `flutter test` 全部通过。

