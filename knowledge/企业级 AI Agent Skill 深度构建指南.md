# 企业级 AI Agent Skill 深度构建指南

## ——从工作流识别到标准 SKILL.md 落地的完全手册

---

# 第一章：认知基石——什么是 Skill？

## 1.1 一句话定义

> **Skill 是一个文件夹，核心是一个 `SKILL.md` 文件，它将专业知识、执行脚本、参考资料打包成模块化的"技能单元"，让 AI Agent 在特定工作场景中像"老同事"一样专业地执行任务。**

更本质地说：Skill 是一份 **写给模型的、结构化的、可长期复用的操作手册**，而非写给人看的文档。

## 1.2 Skill 不是什么——常见认知误区

| 误区 | 正确理解 |
|------|---------|
| Skill = 一段 Prompt | Prompt 是即兴对话，Skill 是**模块化能力单元**，强调稳定复用 |
| Skill 是写给人看的文档 | SKILL.md 是**写给模型的指令**，需结构化、约束行为边界 |
| Skill 就是调 API | Skill 可以不含任何代码，纯 Markdown 指令即可工作 |
| 把所有知识堆进一个 Skill | Skill 遵循"单一职责"，每个 Skill 只干一件事 |
| Skill 是平台特定功能 | 它是**开放标准**，已被 30+ 主流 Agent 工具支持（Claude Code、Cursor、Gemini CLI、OpenClaw、Kimi Code 等） |

## 1.3 Skill 在 AI 技术栈中的定位

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent 技术栈                        │
├──────────┬──────────────────────────────────────────────┤
│  MCP     │  给 Agent 接「外部数据源和工具」（手和脚）       │
├──────────┼──────────────────────────────────────────────┤
│  Skill   │  给 Agent 装「专业知识和 SOP」（大脑和经验）     │
├──────────┼──────────────────────────────────────────────┤
│  Agent   │  调度中枢：理解意图 → 调 Skill → 调 MCP → 执行  │
├──────────┼──────────────────────────────────────────────┤
│  Workflow│  编排多个 Agent/Skill 的复杂流程                │
└──────────┴──────────────────────────────────────────────┘
```

**核心区分：**
- **MCP** 解决的是"Agent 能干什么"（能力边界）
- **Skill** 解决的是"Agent 怎么干"（执行质量）
- 两者配合，才能让 Agent 从"聊天玩具"升级为"生产工具"

## 1.4 Skill 的核心架构原理——渐进式披露（Progressive Disclosure）

这是 Skill 最精髓的设计理念，必须深入理解：

```
三层加载架构：

第一层（始终加载）         → YAML frontmatter 元数据（几百字节）
                             Agent 据此判断是否需要这个 Skill

第二层（按需加载）         → SKILL.md 正文指令（几 KB）
                             当 Agent 判断该 Skill 与当前任务相关时加载

第三层（深度按需加载）     → references/ scripts/ assets/（可达几十万字）
                             Agent 仅在需要某个细节时才读取
```

**为什么这样设计？**

核心原因是 **上下文污染（Context Pollution）**——如果你把所有规则一次性塞给模型：
- Token 消耗飙升
- 模型注意力被无关信息分散
- 执行准确率显著下降
- 多个 Skill 同时加载时相互干扰

渐进式披露让模型在每一刻只看到它需要的信息，就像一个经验丰富的员工——不需要每次做事都把整个公司的制度手册从头到尾翻一遍，而是知道在哪里找到需要的章节。

---

# 第二章：企业级标准的 Skill 是什么？

## 2.1 "能用"与"企业级"的鸿沟

一个"能用的 Skill"只需要一个 SKILL.md 文件就够了。但一个 **企业级标准的 Skill** 需要在以下维度全部达标：

| 维度 | 个人级 Skill | 企业级 Skill |
|------|-------------|-------------|
| **可发现性** | 靠口头告知 | description 语义精准，Agent 自动匹配 |
| **确定性** | 依赖模型即兴发挥 | 关键步骤用脚本固化，行为可预测 |
| **可维护性** | 单文件一改全改 | 模块化分层，独立演进 |
| **可测试性** | 试几次觉得行就用 | 有标准化测试用例和验收标准 |
| **可组合性** | 独立使用 | 可与其他 Skill 协同工作 |
| **版本管理** | 无版本概念 | 语义化版本，变更可追溯 |
| **权限控制** | 无 | 明确的工具权限边界 |
| **知识沉淀** | 写死在指令中 | references 独立管理，可跨 Skill 共享 |
| **容错设计** | 无 | 包含验证节点和回退机制 |
| **团队协作** | 个人使用 | 团队共享，角色分工明确 |

## 2.2 企业级 Skill 的三大设计原则

### 原则一：渐进式披露（已详述）

### 原则二：确定性降级

> **凡是能用脚本做的事，绝不让模型去"写代码做"。**

```
❌ 错误做法：
"请写一段 Python 脚本，把 CSV 文件按日期列排序，然后导出为 Excel"
→ 模型每次生成的代码可能不同，行为不可预测

✅ 正确做法：
"执行 scripts/sort_and_export.py，传入参数：
  --input {csv_path} --output {excel_path} --sort-by date"
→ 同一个脚本永远是同一个行为，确定性 100%
```

### 原则三：强制流程固化

> **在关键节点设置"验证 → 修正 → 再验证"闭环，不允许 Agent 跳过。**

```
Step 3: 生成初稿后，必须执行以下检查清单：
- [ ] 所有数字与源数据一致
- [ ] 没有使用"可能""也许"等模糊表述
- [ ] 格式符合 references/report-template.md
- [ ] 所有图表都有数据来源标注

如有任何一项不通过，返回 Step 2 修正后再继续。
```

---

# 第三章：如何识别并拆解可沉淀的工作流

## 3.1 工作流识别——四象限筛选法

不是所有工作都值得沉淀为 Skill。用以下矩阵筛选：

```
                        高频重复
                           │
              ┌────────────┼────────────┐
              │  ⭐ 优先沉淀  │  考虑自动化  │
              │  （日报周报、 │  （RPA 更合适）│
              │   代码审查、  │            │
              │   会议纪要）  │            │
              ├────────────┼────────────┤
              │  暂不处理    │  高价值沉淀  │
              │  （偶发杂务） │  （尽调报告、 │
              │             │   方案设计）  │
              └────────────┼────────────┘
                           │
                        低频偶发
        ←── 低价值 ──────────────────── 高价值 ──→
```

**优先沉淀的特征：**
1. **高频重复**：每周/每天都做，且步骤基本一致
2. **有明确步骤**：可以写成 Step 1 → Step 2 → Step 3
3. **有判断标准**：某些步骤需要"检查""判断""选择"
4. **有固定产出**：输出格式相对稳定（报告、文档、代码、邮件等）
5. **有经验门槛**：新人需要学习才能做好，老手有隐性知识

## 3.2 工作流拆解——五步解剖法

### 第一步：录制（Record）

像拍纪录片一样，完整记录一次工作流的执行过程：
- 打开哪些工具/系统？
- 查看哪些数据源？
- 做了哪些判断和决策？
- 产出什么中间结果和最终结果？
- 耗时多久？哪些步骤最慢？

### 第二步：抽象（Abstract）

从具体操作中提炼出通用模式：

```
具体操作：打开 JIRA → 筛选 "本周关闭的 BUG" → 复制到 Excel → 写总结
                  ↓ 抽象
通用步骤：采集数据源 → 筛选/过滤 → 结构化处理 → 生成摘要
```

### 第三步：分层（Layer）

将工作流拆分为三层：

| 层次 | 内容 | 对应 Skill 组件 |
|------|------|----------------|
| **决策层** | 何时触发、选择哪条路径 | SKILL.md 指令 |
| **执行层** | 确定性的操作和计算 | scripts/ |
| **知识层** | 参考文档、模板、规范 | references/ assets/ |

### 第四步：标注（Annotate）

为每个步骤标注属性：

- 🤖 **可自动化**：Agent 可独立完成
- 👤 **需人工**：必须人工介入（审批、确认等）
- 🔧 **需工具**：需要调用外部 API/MCP
- ⚠️ **风险点**：容易出错，需要验证
- 🔄 **分支点**：根据不同条件走不同路径

### 第五步：验证（Validate）

让领域专家（实际执行该工作流的人）审查拆解结果：
- 步骤是否遗漏？
- 判断条件是否完整？
- 异常情况是否覆盖？

## 3.3 典型可沉淀工作流清单

| 岗位/场景 | 可沉淀工作流 | Skill 类型 |
|-----------|-------------|-----------|
| 产品经理 | 竞品分析报告生成 | Generator（生成器） |
| 开发工程师 | 代码审查规范执行 | Reviewer（审查器） |
| 测试工程师 | 测试用例自动生成 | Generator + Pipeline |
| 运维工程师 | 故障排查诊断流程 | 决策树 + Tool Wrapper |
| 项目经理 | 周报/月报自动编写 | Generator + Pipeline |
| 数据分析师 | 数据清洗与可视化 | Pipeline + Tool Wrapper |
| 市场运营 | 社交媒体文案批量生成 | Generator |
| HR | 简历筛选与评估 | Reviewer + Inversion |
| 法务 | 合同条款审查 | Reviewer |
| 客服 | 工单分类与回复模板 | 决策树 + Generator |
| 财务 | 费用报销审核 | Reviewer + Pipeline |
| 安全 | 代码安全漏洞扫描 | Reviewer + Tool Wrapper |

---

# 第四章：企业级 Skill 的标准目录与核心要素

## 4.1 标准目录结构

```
skill-name/                          # Skill 根目录（小写 + 连字符命名）
│
├── SKILL.md                         # 【必需】核心指令文件
│   ├── YAML Frontmatter             #   ├─ 元数据（始终加载）
│   │   ├── name:                    #   │   Skill 唯一标识
│   │   ├── description:             #   │   语义触发器（最关键！）
│   │   ├── version:                 #   │   语义化版本号
│   │   ├── author:                  #   │   作者/团队
│   │   ├── license:                 #   │   许可证
│   │   ├── compatibility:           #   │   兼容性要求
│   │   ├── allowed-tools:           #   │   允许使用的工具列表
│   │   └── metadata:                #   │   自定义扩展元数据
│   │       ├── tags:                #   │       标签
│   │       ├── trigger-mode:        #   │       触发方式
│   │       └── priority:            #   │       优先级
│   │
│   └── Markdown Instructions        #   └─ 正文指令（按需加载）
│       ├── ## 触发条件               #       何时激活此 Skill
│       ├── ## 前置准备               #       开始前需要确认的事项
│       ├── ## 执行流程               #       分步骤详细指令
│       ├── ## 输出规范               #       产出物的格式要求
│       ├── ## 异常处理               #       出错时的应对策略
│       └── ## 参考资料引用            #       指向 references/ 的引用
│
├── scripts/                         # 【可选】确定性执行脚本
│   ├── runner.py                    #   主执行脚本
│   ├── validator.sh                 #   验证脚本
│   ├── formatter.py                 #   格式化脚本
│   └── requirements.txt             #   依赖声明
│
├── references/                      # 【可选】按需加载的参考文档
│   ├── api-spec.md                  #   API 规范文档
│   ├── style-guide.md               #   风格指南
│   ├── checklist.md                 #   检查清单
│   ├── faq.md                       #   常见问题
│   └── examples/                    #   示例集合
│       ├── good-example.md
│       └── bad-example.md
│
└── assets/                          # 【可选】静态资源文件
    ├── templates/                   #   输出模板
    │   └── report-template.md
    ├── configs/                     #   配置文件
    └── media/                       #   图片、字体等
```

## 4.2 各核心要素详解

### 4.2.1 SKILL.md——灵魂文件

SKILL.md 是整个 Skill 的核心，它做两件事：
1. **告诉 Agent 什么时候该用我**（通过 frontmatter 的 description）
2. **告诉 Agent 拿到我之后该怎么做**（通过 Markdown 正文指令）

### 4.2.2 scripts/——确定性执行层

**核心原则：凡是行为必须 100% 一致的，都用脚本而非指令。**

```python
# scripts/generate_report.py
"""
用途：根据输入数据生成标准化周报
调用方式：python scripts/generate_report.py --input data.json --output report.md
"""
import argparse
import json
from datetime import datetime

def generate_report(input_path: str, output_path: str):
    with open(input_path) as f:
        data = json.load(f)
    
    report = f"""# 周报 - {datetime.now().strftime('%Y-%m-%d')}
    
## 本周完成
{chr(10).join(f'- {item}' for item in data['completed'])}

## 下周计划  
{chr(10).join(f'- {item}' for item in data['planned'])}

## 风险与阻塞
{chr(10).join(f'- {item}' for item in data['risks'])}
"""
    with open(output_path, 'w') as f:
        f.write(report)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--input', required=True)
    parser.add_argument('--output', required=True)
    args = parser.parse_args()
    generate_report(args.input, args.output)
```

### 4.2.3 references/——知识库层

references 中的文件 **不会被自动加载**，只有当 Agent 在执行过程中主动读取时才会注入上下文。这意味着：

- 可以将大量参考文档放在这里，不用担心撑爆上下文
- 用 `{{load: references/xxx.md}}` 语法或自然语言指引 Agent 按需读取
- 每个 reference 文件应该自包含，有清晰的标题和目录

### 4.2.4 assets/——静态资源层

存放模板文件、配置样例、图片素材等。Agent 在生成输出时可以引用这些资源。

## 4.3 YAML Frontmatter 所有字段规范

```yaml
---
# ═══ 必填字段 ═══
name: weekly-report-generator
  # 规则：小写字母 + 数字 + 连字符，全局唯一
  # 作用：Skill 的唯一标识符，用于引用和调用

description: >
  根据项目管理工具（Jira/飞书）中的任务数据，自动生成团队周报。
  当用户提到"写周报""生成周报""本周总结""weekly report"时使用此技能。
  支持自动采集已完成任务、进行中的任务、以及下周计划。
  # 规则：150-300 字符，必须包含触发关键词和使用场景
  # 作用：Agent 据此判断是否加载此 Skill——这是 Skill 被发现的唯一途径！

# ═══ 推荐字段 ═══
version: 1.2.0
  # 语义化版本：主版本.次版本.修订号

author: platform-team
  # 作者或负责团队

license: Apache-2.0
  # 许可证类型

compatibility: 需要 Python 3.10+，需安装 jinja2
  # 运行环境和依赖声明

allowed-tools:
  - Read           # 允许读取文件
  - Write          # 允许写入文件
  - Bash           # 允许执行命令
  - WebFetch       # 允许网络请求
  # 明确权限边界，防止 Agent 越权操作

# ═══ 扩展元数据 ═══
metadata:
  tags: [report, weekly, team-management]
  trigger-mode: auto    # auto（自动匹配）| manual（手动 /command）| both
  priority: high        # 当多个 Skill 匹配时的优先级
  scope: team           # personal | team | organization
  last-reviewed: 2026-06-15
  maintainer: zhangsan@company.com
---
```

---

# 第五章：SKILL.md 正文——五种核心设计模式

## 5.1 Google 总结的五大设计模式

Google Cloud Tech 的 Shubham Saboo 和 Lavini Nigam 通过研究整个生态中的 Skill 设计，总结出了五种核心模式：

### 模式一：Tool Wrapper（工具包装器）

**解决问题**：让 Agent 瞬间成为某个特定工具/库/框架的专家

```markdown
---
name: docker-expert
description: >
  Docker 容器化部署专家。当用户需要编写 Dockerfile、docker-compose、
  处理容器网络、卷挂载、多阶段构建等问题时使用。
---

# Docker 专家技能

## 核心规则
1. 始终使用多阶段构建减小镜像体积
2. 非 root 用户运行（创建 appuser）
3. 利用 .dockerignore 排除无关文件
4. 生产环境使用 alpine/slim 基础镜像

## Dockerfile 最佳实践
- COPY 顺序：先依赖文件，再源代码（利用缓存层）
- 合并 RUN 指令减少层数
- 使用 HEALTHCHECK 指令
- 暴露端口使用 EXPOSE 声明

## 常见陷阱
- 不要在 RUN 中存储密钥
- 避免使用 `latest` 标签
- 注意 COPY 和 ADD 的区别

## 参考文档
如需查看完整的 Docker 安全规范，读取 references/docker-security.md
```

**适用场景**：框架规范、API 使用指南、工具最佳实践

### 模式二：Generator（生成器）

**解决问题**：基于可复用模板生成结构化文档

```markdown
---
name: api-doc-generator
description: >
  API 文档生成器。当用户需要为 REST API 或 GraphQL 接口生成技术文档时使用。
  支持 OpenAPI 3.0 格式输出。
---

# API 文档生成器

## 执行流程

### Step 1: 信息收集
向用户确认以下信息（如未提供则主动询问）：
- API 端点列表
- 请求/响应示例
- 认证方式
- 错误码定义

### Step 2: 结构生成
按以下模板组织文档结构：
1. 概述（一句话说明 API 用途）
2. 认证方式
3. 端点列表（按资源分组）
4. 每个端点包含：URL、Method、参数表、请求示例、响应示例、错误码
5. 通用错误码表
6. SDK 示例（可选）

### Step 3: 内容填充
- 读取 assets/api-doc-template.md 获取模板
- 根据 Step 1 收集的信息填充模板
- 所有示例必须可直接复制运行

### Step 4: 质量检查
- [ ] 所有端点都有请求和响应示例
- [ ] 参数类型与示例值一致
- [ ] 错误码覆盖完整
- [ ] Markdown 格式正确

## 输出格式
使用 OpenAPI 3.0 YAML 格式 + 人类可读的 Markdown 双输出
```

**适用场景**：报告生成、文档生成、代码脚手架、邮件模板

### 模式三：Reviewer（审查器）

**解决问题**：按检查清单逐项审查并给出分级结论

```markdown
---
name: code-review
description: >
  代码审查技能。当用户提交代码请求 review、PR 审查、代码质量检查时使用。
  覆盖安全性、性能、可读性、架构四个维度。
---

# 代码审查技能

## 审查流程

### Phase 1: 快速扫描
先整体浏览代码变更，理解意图和范围。不做评判，只做理解。

### Phase 2: 逐维度审查
按以下四个维度依次检查，每个维度独立评分（1-5）：

#### 2.1 安全性 (Security)
读取 references/security-checklist.md 获取完整检查项。核心项：
- SQL 注入风险
- XSS 漏洞
- 敏感信息硬编码
- 权限校验缺失

#### 2.2 性能 (Performance)
- N+1 查询
- 内存泄漏风险
- 不必要的循环嵌套
- 缓存使用合理性

#### 2.3 可读性 (Readability)
- 命名规范
- 函数长度（建议 < 30 行）
- 注释充分性
- 代码重复

#### 2.4 架构 (Architecture)
- 职责单一原则
- 依赖方向合理性
- 接口设计一致性

### Phase 3: 生成审查报告

输出格式：
## 审查总结
总体评分：X/5
关键发现：[数量]

## 严重问题（必须修复）
- 🔴 [文件:行号] 问题描述 → 建议修复方式

## 改进建议（建议修复）
- 🟡 [文件:行号] 问题描述 → 建议修复方式

## 亮点（值得学习）
- 🟢 [文件:行号] 好的做法

## 通过建议
[ ] 可以直接合并 / [ ] 需要修改后重新审查
```

**适用场景**：代码审查、文档校对、合规检查、安全审计

### 模式四：Inversion（反转）

**解决问题**：Agent 固有的"猜测并立即生成"倾向，导致输出偏离需求

```markdown
---
name: requirement-analyst
description: >
  需求分析专家。当用户描述一个模糊的项目需求、产品构想、或业务问题时使用。
  核心能力：通过结构化提问，帮助用户澄清需求，而非急于给出方案。
---

# 需求分析技能

## 核心原则
⚠️ 在充分理解需求之前，绝对不要给出解决方案！
你的角色是"需求顾问"而非"方案生成器"。

## 执行流程

### Step 1: 理解阶段（必须完成，不可跳过）
向用户提出以下结构化问题（根据上下文选择最相关的 5-8 个）：

**关于目标：**
- 这个需求要解决什么核心问题？
- 成功的标准是什么？如何衡量？
- 有没有参考案例或竞品？

**关于约束：**
- 时间约束是什么？有没有 deadline？
- 预算/资源约束？
- 技术栈有没有限制？

**关于用户：**
- 最终用户是谁？
- 用户的核心痛点是什么？
- 用户目前是如何解决这个问题的？

**关于风险：**
- 最大的不确定性在哪里？
- 如果失败了，最可能的原因是什么？

### Step 2: 确认阶段
将收集到的信息整理为结构化摘要，请用户确认：
- 我的理解是否准确？
- 有没有遗漏的关键信息？
- 优先级是否正确？

### Step 3: 方案阶段（仅在确认后开始）
基于确认后的需求，提供 2-3 个方案选项：
- 每个方案包含：核心思路、优势、劣势、预估工作量
- 给出推荐方案及理由
```

**适用场景**：需求分析、方案设计、问题诊断、咨询类工作流

### 模式五：Pipeline（流水线）

**解决问题**：强制 Agent 按顺序执行复杂的多步骤流程

```markdown
---
name: data-analysis-pipeline
description: >
  数据分析流水线。当用户提供原始数据并要求进行分析、生成洞察报告时使用。
  支持 CSV、Excel、JSON 格式输入。
---

# 数据分析流水线

## 重要约束
这是一个严格的顺序流水线。每个阶段必须完成并通过检查点后，
才能进入下一阶段。不允许跳过任何阶段。

## Pipeline 阶段

### Phase 1: 数据摄入与理解 📥
1. 读取用户提供的数据文件
2. 输出数据概况：
   - 行数、列数、列名及类型
   - 缺失值统计
   - 基础统计描述
3. **检查点**：向用户确认数据理解是否正确
   → 不通过则返回修正

### Phase 2: 数据清洗 🧹
1. 处理缺失值（记录处理策略）
2. 处理异常值
3. 数据类型转换
4. **检查点**：输出清洗报告，用户确认
   → 不通过则返回修正

### Phase 3: 探索性分析 🔍
1. 单变量分析（分布、趋势）
2. 多变量分析（相关性、交叉分析）
3. 时间维度分析（如有时间列）
4. **检查点**：输出分析摘要，用户选择深入方向
   → 用户可选择 2-3 个方向深入

### Phase 4: 洞察生成 💡
1. 基于 Phase 3 结果提炼 3-5 个核心洞察
2. 每个洞察包含：发现、数据支撑、业务含义、建议行动
3. **检查点**：用户确认洞察是否有价值
   → 不通过则补充分析

### Phase 5: 报告输出 📊
1. 读取 assets/report-template.md
2. 按模板生成完整分析报告
3. 包含所有可视化描述和数据表格
4. 执行 scripts/validate_report.py 做最终格式验证

## 异常处理
- 数据文件无法读取 → 提示用户检查文件格式
- 数据量过大 → 建议采样分析，说明采样策略
- 分析过程中断 → 从最近的检查点恢复
```

**适用场景**：数据分析、报告生成、项目交付、审计流程

## 5.2 模式组合使用

实际企业场景中，一个复杂的 Skill 往往是多种模式的组合：

```
竞品分析 Skill = Inversion（先收集需求）
                + Pipeline（按阶段分析）
                + Generator（生成报告）
                + Tool Wrapper（调用数据源 API）
```

---

# 第六章：实战演练——精心打磨一款标准 SKILL.md

## 6.1 场景选择：团队周报自动生成

**背景**：某研发团队每周五下午，项目经理需要花 2-3 小时手动编写团队周报。流程如下：
1. 打开 Jira 查看本周关闭的任务
2. 打开 GitLab 查看本周合并的 MR
3. 整理各组员的进度汇报（飞书文档）
4. 汇总到固定格式的周报模板中
5. 撰写下周计划
6. 标注风险和阻塞项
7. 发送给管理层

## 6.2 工作流拆解

```
┌──────────────────────────────────────────────────────────┐
│  团队周报生成工作流                                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  [Step 1] 信息收集（Inversion 模式）                       │
│    ├── 确认周报周期（默认本周一至周五）                      │
│    ├── 确认团队成员列表                                    │
│    └── 确认重点项目/里程碑                                  │
│         │                                                │
│         ▼                                                │
│  [Step 2] 数据采集（Tool Wrapper 模式）                    │
│    ├── 🤖 从项目管理工具获取已完成任务                      │
│    ├── 🤖 从代码仓库获取提交和合并记录                      │
│    └── 👤 收集团队成员自述进展（如有）                      │
│         │                                                │
│         ▼                                                │
│  [Step 3] 数据整合（Pipeline 模式）                        │
│    ├── 去重合并（同一工作在多个系统有记录）                  │
│    ├── 分类归档（按项目/模块分组）                          │
│    ├── ⚠️ 标记异常（进度偏差、延期风险）                    │
│    └── 🔧 执行 scripts/aggregate_data.py                  │
│         │                                                │
│         ▼                                                │
│  [Step 4] 报告生成（Generator 模式）                       │
│    ├── 读取 assets/weekly-report-template.md               │
│    ├── 填充数据和文字描述                                  │
│    ├── 撰写下周计划（基于任务 backlog）                     │
│    └── 生成风险摘要                                       │
│         │                                                │
│         ▼                                                │
│  [Step 5] 质量验证（Reviewer 模式）                        │
│    ├── 🔧 执行 scripts/validate_report.py                 │
│    ├── 检查清单逐项验证                                    │
│    └── 👤 请用户确认最终版本                               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## 6.3 完整 SKILL.md 编写

以下是这个 Skill 的完整文件内容：

---

### 📄 `weekly-report/SKILL.md`

```markdown
---
name: team-weekly-report
description: >
  团队周报自动生成器。当用户提到"写周报""生成周报""本周总结""团队周报"
  "weekly report""周五报告"时使用此技能。
  能从项目管理工具、代码仓库、协作文档中自动采集数据，
  按标准模板生成结构化的团队周报，包含本周完成、下周计划、风险预警。
version: 1.2.0
author: platform-engineering
license: MIT
compatibility: 需要 Python 3.10+，需要 Jira/GitLab API 访问权限
allowed-tools:
  - Read
  - Write
  - Bash
  - WebFetch
metadata:
  tags: [report, weekly, team, project-management]
  trigger-mode: both
  priority: high
  scope: team
  last-reviewed: 2026-06-15
---

# 团队周报自动生成器

## 角色定义
你是一位经验丰富的项目管理助手，负责帮助团队负责人高效生成结构化周报。
你的输出将直接提交给管理层，因此必须专业、准确、简洁。

## 触发条件
当用户请求生成团队周报、本周总结、或周期性工作汇报时激活此技能。

## 前置准备
在开始之前，确认以下信息（如用户未提供则主动询问）：

| 信息项 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| 报告周期 | 是 | 本周一至周五 | 起止日期 |
| 团队名称 | 是 | 无 | 团队标识 |
| 成员列表 | 是 | 无 | 参与本周工作的成员 |
| 数据来源 | 否 | Jira + GitLab | 数据采集渠道 |
| 重点项目 | 否 | 无 | 需要特别关注的项目/里程碑 |
| 上期遗留 | 否 | 无 | 上周未完成需跟踪的事项 |

## 执行流程

### Phase 1: 需求确认 ⏱️
1. 根据"前置准备"表格收集必要信息
2. 整理为确认摘要，请用户确认
3. **⚠️ 检查点：用户确认后方可进入 Phase 2**

### Phase 2: 数据采集 📥

#### 2.1 从项目管理工具采集
- 获取报告周期内状态变更为"已完成"/"已关闭"的任务
- 获取当前"进行中"的任务及其进度
- 获取新建但未开始的任务
- 对每条任务记录：编号、标题、负责人、状态、耗时

#### 2.2 从代码仓库采集
- 获取报告周期内合并的 MR/PR 列表
- 获取代码提交统计（按人）
- 对每个 MR 记录：编号、标题、作者、变更文件数、合并时间

#### 2.3 数据合并
执行聚合脚本：
```bash
python scripts/aggregate_data.py \
  --tasks tasks.json \
  --merge-requests mrs.json \
  --members members.json \
  --output aggregated.json
```

**⚠️ 检查点：向用户展示采集到的数据概况，确认是否有遗漏**

### Phase 3: 内容组织 📝

#### 3.1 分类归档
将采集到的工作项按以下维度分组：
1. **按项目/模块分组**（主维度）
2. **按完成状态分组**：已完成 / 进行中 / 延期 / 新增
3. **按重要性标注**：关键里程碑 / 常规迭代 / 技术优化 / Bug修复

#### 3.2 进度分析
- 对比计划与实际完成情况
- 识别延期项并分析原因
- 标记超预期完成的亮点

#### 3.3 下周计划生成
基于以下信息生成下周计划：
- 当前"进行中"但未完成的任务
- Backlog 中优先级最高的待办
- 即将到来的里程碑和 deadline

### Phase 4: 报告生成 📊

1. 读取模板文件：`assets/weekly-report-template.md`
2. 按模板结构填充内容：

#### 报告结构要求：

**一、本周概要**（3-5 句话的摘要，管理层可只看这段）
- 必须包含：关键数字（完成 X 项任务、合并 Y 个 MR）
- 必须包含：最重要的 1-2 个进展

**二、已完成工作**（按项目分组，表格形式）
| 项目 | 工作项 | 负责人 | 备注 |
|------|--------|--------|------|

**三、进行中工作**（标注预计完成时间和当前进度）

**四、风险与阻塞**（⚠️ 最重要的一段）
- 每个风险项包含：描述、影响范围、当前状态、建议措施
- 使用严重度标签：🔴 严重 / 🟡 中等 / 🟢 轻微

**五、下周计划**（按优先级排序）

**六、关键指标**（可选，如团队有度量指标）
- 本周交付速率
- Bug 修复率
- 代码质量指标

### Phase 5: 质量验证 ✅

执行验证脚本：
```bash
python scripts/validate_report.py --input report.md --strict
```

然后逐项检查：
- [ ] 所有数字与源数据一致（无虚构数据）
- [ ] 无模糊表述（"大概""可能""差不多"等）
- [ ] 风险项都有对应的建议措施
- [ ] 下周计划具体可执行（不是"继续推进 XX"）
- [ ] 格式符合模板规范
- [ ] 总长度控制在 2 页以内（约 1500-2500 字）

**⚠️ 任何一项不通过，返回 Phase 4 修正**

### Phase 6: 最终确认 👤
将生成的周报完整呈现给用户，并询问：
1. 内容是否准确？有需要修改的地方吗？
2. 是否需要补充某些未采集到的信息？
3. 确认后输出最终版本

## 输出规范
- 格式：Markdown
- 语言：专业简洁的中文，避免口语化
- 数据：所有数据必须有来源，严禁编造
- 长度：1500-2500 字（约 2 页）
- 风格：事实驱动，结论先行，数据支撑

## 异常处理

| 异常情况 | 处理方式 |
|----------|---------|
| 无法连接项目管理工具 | 提示用户手动提供任务列表或检查 API 配置 |
| 数据为空（本周无记录） | 提醒用户确认数据源是否正确，或手动输入 |
| 数据冲突（同一工作多处记录） | 以项目管理工具为准，代码仓库作为补充 |
| 用户提供的信息不完整 | 明确列出缺失项，用"[待补充]"标记 |

## 参考文档
- 如需查看报告模板详细说明：读取 `references/template-guide.md`
- 如需查看历史周报示例：读取 `references/example-reports.md`
- 如需查看团队 OKR 以对齐重点：读取 `references/team-okr.md`
```

---

### 📄 `weekly-report/scripts/aggregate_data.py`

```python
"""
周报数据聚合脚本
将来自不同数据源的工作项合并为统一视图
"""
import argparse
import json
from collections import defaultdict
from datetime import datetime

def aggregate(tasks_path: str, mrs_path: str, members_path: str) -> dict:
    with open(tasks_path) as f:
        tasks = json.load(f)
    with open(mrs_path) as f:
        mrs = json.load(f)
    with open(members_path) as f:
        members = json.load(f)

    # 按项目分组
    by_project = defaultdict(lambda: {
        "completed": [], "in_progress": [], "blocked": []
    })

    for task in tasks:
        project = task.get("project", "未分类")
        status = task.get("status", "unknown")
        item = {
            "id": task["id"],
            "title": task["title"],
            "assignee": task.get("assignee", "未分配"),
            "type": task.get("type", "task"),
        }
        if status in ("done", "closed"):
            by_project[project]["completed"].append(item)
        elif status == "in_progress":
            item["progress"] = task.get("progress", 0)
            by_project[project]["in_progress"].append(item)
        elif status == "blocked":
            item["reason"] = task.get("block_reason", "")
            by_project[project]["blocked"].append(item)

    # 合并 MR 信息
    mr_by_author = defaultdict(list)
    for mr in mrs:
        mr_by_author[mr.get("author", "unknown")].append({
            "id": mr["id"],
            "title": mr["title"],
            "files_changed": mr.get("files_changed", 0),
        })

    return {
        "generated_at": datetime.now().isoformat(),
        "projects": dict(by_project),
        "code_contributions": dict(mr_by_author),
        "team_members": members,
        "summary": {
            "total_completed": sum(
                len(p["completed"]) for p in by_project.values()
            ),
            "total_in_progress": sum(
                len(p["in_progress"]) for p in by_project.values()
            ),
            "total_blocked": sum(
                len(p["blocked"]) for p in by_project.values()
            ),
            "total_mrs": len(mrs),
        }
    }

def main():
    parser = argparse.ArgumentParser(description="聚合周报数据")
    parser.add_argument("--tasks", required=True, help="任务数据 JSON")
    parser.add_argument("--merge-requests", required=True, help="MR 数据 JSON")
    parser.add_argument("--members", required=True, help="团队成员 JSON")
    parser.add_argument("--output", required=True, help="输出文件路径")
    args = parser.parse_args()

    result = aggregate(args.tasks, args.merge_requests, args.members)
    with open(args.output, "w") as f:
        json.dump(result, f, ensure_ascii=False, indent=2)
    print(f"✅ 数据聚合完成，输出至 {args.output}")
    print(f"   已完成: {result['summary']['total_completed']} 项")
    print(f"   进行中: {result['summary']['total_in_progress']} 项")
    print(f"   阻塞:   {result['summary']['total_blocked']} 项")

if __name__ == "__main__":
    main()
```

---

### 📄 `weekly-report/scripts/validate_report.py`

```python
"""
周报质量验证脚本
检查生成的周报是否符合规范
"""
import argparse
import re
import sys

FUZZY_WORDS = ["大概", "可能", "差不多", "也许", "应该", "尽量", "争取"]
MIN_LENGTH = 1500
MAX_LENGTH = 2500

def validate(report_path: str, strict: bool = False) -> list:
    with open(report_path) as f:
        content = f.read()

    issues = []

    # 1. 长度检查
    char_count = len(content)
    if char_count < MIN_LENGTH:
        issues.append(f"🟡 报告偏短（{char_count}字），建议补充细节")
    elif char_count > MAX_LENGTH:
        issues.append(f"🟡 报告偏长（{char_count}字），建议精简")

    # 2. 模糊表述检查
    for word in FUZZY_WORDS:
        if word in content:
            issues.append(f"🔴 发现模糊表述：'{word}'，请替换为具体描述")

    # 3. 必要章节检查
    required_sections = ["本周概要", "已完成工作", "风险与阻塞", "下周计划"]
    for section in required_sections:
        if section not in content:
            issues.append(f"🔴 缺少必要章节：'{section}'")

    # 4. 风险项是否有建议措施
    risk_section = content.split("风险与阻塞")[-1].split("##")[0] if "风险与阻塞" in content else ""
    risk_items = re.findall(r"[-*] .+", risk_section)
    for item in risk_items:
        if "建议" not in item and "措施" not in item and "方案" not in item:
            issues.append(f"🟡 风险项缺少建议措施：'{item[:40]}...'")

    # 5. 数据检查（是否包含数字）
    numbers = re.findall(r"\d+", content)
    if len(numbers) < 5:
        issues.append("🟡 报告中数字引用偏少，请确认数据完整性")

    return issues

def main():
    parser = argparse.ArgumentParser(description="验证周报质量")
    parser.add_argument("--input", required=True, help="周报文件路径")
    parser.add_argument("--strict", action="store_true", help="严格模式")
    args = parser.parse_args()

    issues = validate(args.input, args.strict)

    if not issues:
        print("✅ 周报质量检查全部通过！")
        sys.exit(0)
    else:
        print(f"⚠️ 发现 {len(issues)} 个问题：")
        for issue in issues:
            print(f"   {issue}")
        if args.strict:
            sys.exit(1)

if __name__ == "__main__":
    main()
```

---

### 📄 `weekly-report/assets/weekly-report-template.md`

```markdown
# 📋 {team_name} 周报

> 报告周期：{start_date} ~ {end_date}
> 撰写人：{author}
> 生成时间：{generated_at}

---

## 一、本周概要

{summary_text}

**关键数字：** 完成 {completed_count} 项任务 | 合并 {mr_count} 个 MR | 
{blocked_count} 项阻塞待解决

---

## 二、已完成工作

{for_each_project}
### {project_name}

| 工作项 | 负责人 | 类型 | 备注 |
|--------|--------|------|------|
| {task_title} | {assignee} | {type} | {note} |

{end_for}

---

## 三、进行中工作

| 工作项 | 负责人 | 进度 | 预计完成 | 备注 |
|--------|--------|------|----------|------|
| {task_title} | {assignee} | {progress}% | {eta} | {note} |

---

## 四、风险与阻塞

| 严重度 | 描述 | 影响 | 状态 | 建议措施 |
|--------|------|------|------|---------|
| 🔴/🟡/🟢 | {description} | {impact} | {status} | {action} |

---

## 五、下周计划

| 优先级 | 工作项 | 负责人 | 预计产出 |
|--------|--------|--------|---------|
| P0/P1/P2 | {task} | {assignee} | {deliverable} |

---

## 六、关键指标

| 指标 | 本周 | 上周 | 趋势 |
|------|------|------|------|
| 任务完成率 | {rate}% | {last_rate}% | ↑/↓/→ |
| Bug 修复数 | {bugs} | {last_bugs} | ↑/↓/→ |
| 代码行数变更 | +{add}/-{del} | - | - |
```

---

### 📄 `weekly-report/references/template-guide.md`

```markdown
# 周报模板使用指南

## 字段说明

### 本周概要
- 控制在 3-5 句话
- 第一句必须是本周最重要的成果
- 必须包含至少 2 个具体数字

### 风险严重度定义
- 🔴 严重：影响项目进度或客户交付，需要管理层介入
- 🟡 中等：影响团队效率，可在团队层面解决
- 🟢 轻微：暂时不影响，但需持续关注

### 优先级定义
- P0：下周必须完成，否则影响里程碑
- P1：下周应该完成
- P2：下周计划完成，可适当延后

## 常见问题
Q: 某项工作跨周怎么办？
A: 在"进行中"记录进度，在"下周计划"中继续跟踪

Q: 非项目性工作（如培训、面试）怎么记录？
A: 归入"其他工作"分类
```

---

## 6.4 文件总览

```
weekly-report/
├── SKILL.md                              # 核心指令（~200 行）
├── scripts/
│   ├── aggregate_data.py                 # 数据聚合（确定性操作）
│   ├── validate_report.py                # 质量验证（确定性检查）
│   └── requirements.txt                  # 依赖声明
├── references/
│   ├── template-guide.md                 # 模板字段详细说明
│   ├── example-reports.md                # 历史优秀周报示例
│   └── team-okr.md                       # 团队 OKR（用于对齐重点）
└── assets/
    └── weekly-report-template.md         # 周报结构模板
```

---

# 第七章：深度进阶——从"能用"到"卓越"的关键技巧

## 7.1 description 是灵魂——如何写好语义触发器

description 是 Skill 被 Agent 发现的 **唯一途径**。写不好，Skill 就等于不存在。

### ❌ 糟糕的 description

```yaml
description: 帮助生成报告
# 问题：太模糊，Agent 无法判断何时匹配
```

### ⚠️ 一般的 description

```yaml
description: 生成团队周报，包含本周完成的工作和下周计划
# 问题：只有功能描述，缺少触发关键词
```

### ✅ 优秀的 description

```yaml
description: >
  团队周报自动生成器。当用户提到"写周报""生成周报""本周总结"
  "团队周报""weekly report""周五报告"时使用此技能。
  能从项目管理工具、代码仓库、协作文档中自动采集数据，
  按标准模板生成结构化的团队周报。
# 优点：包含 6 个触发关键词 + 数据来源说明 + 输出格式说明
```

### 写好 description 的公式

```
[做什么] + [什么时候触发（关键词列表）] + [输入是什么] + [输出是什么]
```

## 7.2 指令写作的"黄金法则"

### 法则一：具体化，不要抽象化

```
❌ "请生成一份高质量的报告"
✅ "生成一份 Markdown 格式的报告，长度 1500-2500 字，
   包含以下章节：摘要、已完成工作（表格形式）、
   风险与阻塞（含严重度标签）、下周计划（含优先级）"
```

### 法则二：用约束代替自由

```
❌ "你可以自由发挥来组织报告内容"
✅ "报告必须遵守以下约束：
   - 不使用'可能''大概'等模糊词
   - 每个风险项必须附带建议措施
   - 数据必须来自 Phase 2 的采集结果，严禁编造"
```

### 法则三：提供正反例

```markdown
## 好的风险描述（学习这个）
"🔴 Jira API 响应时间从 200ms 升至 2000ms，影响数据采集效率。
 已与运维团队沟通，预计周三修复。临时方案：改用批量导出。"

## 差的风险描述（避免这个）
"🟡 系统有点慢，可能影响进度。"
```

### 法则四：明确告诉 Agent 什么不能做

```markdown
## 禁止行为
- ❌ 不得编造任何不存在的数据或工作项
- ❌ 不得在没有用户确认的情况下发送报告
- ❌ 不得跳过 Phase 5 的质量验证
- ❌ 不得在报告中包含团队成员的个人绩效评语
```

## 7.3 脚本调用的最佳实践

### 模式：SKILL.md 中如何引用脚本

```markdown
### Step 3: 数据聚合
执行以下命令进行数据聚合（不要自己写代码，直接调用脚本）：

```bash
python scripts/aggregate_data.py \
  --tasks {tasks_json_path} \
  --merge-requests {mrs_json_path} \
  --members {members_json_path} \
  --output aggregated.json
```

**预期输出：**
- 成功时：打印统计摘要，生成 aggregated.json
- 失败时：打印错误信息，此时请提示用户检查输入数据格式

**⚠️ 重要：如果脚本执行失败，不要尝试自己重写脚本逻辑，
而是将错误信息完整呈现给用户，请求协助。**
```

## 7.4 处理 Skill 之间的依赖与组合

当一个复杂工作流需要多个 Skill 协作时：

```
skill-orchestrator/           # 编排层 Skill
├── SKILL.md                  # 调度其他 Skill
│
│   "Phase 1: 使用 data-collection Skill 采集数据"
│   "Phase 2: 使用 data-analysis Skill 分析数据"  
│   "Phase 3: 使用 report-generator Skill 生成报告"
│
├── data-collection/          # 子 Skill 1
│   └── SKILL.md
├── data-analysis/            # 子 Skill 2
│   └── SKILL.md
└── report-generator/         # 子 Skill 3
└── SKILL.md
```

在编排 Skill 中引用子 Skill：

```markdown
## 执行流程

### Phase 1: 数据采集
加载并使用 data-collection 技能完成数据采集工作。
将结果保存为 data/raw.json，供后续阶段使用。

### Phase 2: 数据分析
加载并使用 data-analysis 技能，输入为 data/raw.json。
将分析结果保存为 data/analysis.json。
```

## 7.5 Skill 的存储与分发策略

| 级别 | 存储位置 | 适用场景 | 共享范围 |
|------|---------|---------|---------|
| **个人级** | `~/.claude/skills/` | 个人通用习惯 | 仅自己 |
| **项目级** | `.claude/skills/`（项目根目录下） | 项目特定规范 | 项目参与者 |
| **团队级** | 共享 Git 仓库 + 软链接 | 团队统一标准 | 团队成员 |
| **组织级** | 内部 Skill Registry | 企业知识沉淀 | 全组织 |

## 7.6 版本管理与变更控制

```yaml
# SKILL.md frontmatter 中的版本管理
version: 2.1.0
# 主版本：不兼容的结构性变更（如流程重排、字段删除）
# 次版本：新增功能或步骤（向后兼容）
# 修订号：修正错误、优化措辞

metadata:
  changelog:
    - version: 2.1.0
      date: 2026-06-15
      changes: "新增 Phase 5 质量验证步骤"
    - version: 2.0.0
      date: 2026-05-20
      changes: "重构为 Pipeline 模式，增加检查点机制"
    - version: 1.0.0
      date: 2026-04-01
      changes: "初始版本"
```

## 7.7 测试与验证策略

### 创建标准化测试用例

```
weekly-report/
├── tests/
│   ├── test-data/                    # 测试用模拟数据
│   │   ├── sample-tasks.json
│   │   ├── sample-mrs.json
│   │   └── sample-members.json
│   ├── expected-output/              # 期望的输出结果
│   │   └── expected-report.md
│   └── test-cases.md                 # 测试用例文档
```

### 测试用例设计

```markdown
# test-cases.md

## 测试用例 1：正常流程
输入：完整的模拟数据（20 条任务、10 个 MR、5 名成员）
期望：生成符合模板的周报，所有章节齐全

## 测试用例 2：数据缺失
输入：只有任务数据，无 MR 数据
期望：报告中代码贡献部分显示"[数据缺失]"，不报错

## 测试用例 3：大量数据
输入：100+ 条任务
期望：报告长度仍在 2500 字以内，合理精简

## 测试用例 4：边界情况
输入：本周无任何工作记录
期望：提示用户确认数据源，而非生成空报告
```

---

# 第八章：企业级 Skill 治理与生命周期管理

## 8.1 Skill 生命周期

```
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│  识别   │───→│  设计   │───→│  开发   │───→│  测试   │───→│  发布   │
│ Identify│    │ Design  │    │Develop │    │  Test   │    │Publish │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘
                                                          │
     ┌────────────────────────────────────────────────────┘
     ▼
┌────────┐    ┌────────┐    ┌────────┐
│  运营   │───→│  迭代   │───→│  退役   │
│Operate │    │Iterate │    │Retire  │
└────────┘    └────────┘    └────────┘
```

## 8.2 团队协作模式

```
角色分工：
┌─────────────────────────────────────────────────┐
│ Skill Owner（技能负责人）                         │
│  - 负责 Skill 的整体质量和演进                     │
│  - 审核变更请求                                   │
│  - 处理使用反馈                                   │
├─────────────────────────────────────────────────┤
│ Domain Expert（领域专家）                         │
│  - 提供工作流的专业知识                            │
│  - 验证 Skill 内容的准确性                         │
│  - 参与测试用例设计                                │
├─────────────────────────────────────────────────┤
│ Skill Developer（技能开发者）                     │
│  - 编写 SKILL.md 和脚本                           │
│  - 实现技术层面的逻辑                              │
│  - 维护 references 和 assets                      │
├─────────────────────────────────────────────────┤
│ End User（终端用户）                              │
│  - 日常使用 Skill                                 │
│  - 提供使用反馈                                   │
│  - 报告异常情况                                   │
└─────────────────────────────────────────────────┘
```

## 8.3 质量度量指标

| 指标 | 说明 | 目标值 |
|------|------|--------|
| **触发准确率** | Agent 正确识别并加载 Skill 的比例 | > 95% |
| **执行成功率** | Skill 执行完成且输出可用的比例 | > 90% |
| **人工干预率** | 需要用户手动修正的比例 | < 20% |
| **用户满意度** | 用户对输出质量的评分 | > 4/5 |
| **平均执行时间** | 从触发到输出的耗时 | 视场景而定 |
| **Token 消耗** | 单次执行的 Token 用量 | 持续优化 |

## 8.4 持续优化循环

```
每次使用后记录：
1. 哪些步骤执行顺利？
2. 哪些步骤 Agent 理解偏差？
3. 用户做了哪些手动修正？
4. 有无新增的边界情况？

定期 Review（建议每月一次）：
1. 分析使用日志和反馈
2. 更新 SKILL.md 指令
3. 优化脚本逻辑
4. 补充 references 知识库
5. 更新测试用例
```

---

# 第九章：避坑指南——常见错误与解决方案

## 9.1 十大常见错误

| # | 错误 | 原因 | 解决方案 |
|---|------|------|---------|
| 1 | Skill 从未被触发 | description 太模糊或缺少关键词 | 丰富触发词，覆盖用户可能的表述方式 |
| 2 | Agent 跳过关键步骤 | 指令中缺少强制约束 | 使用"必须""不允许跳过"等强制语言 + 检查点 |
| 3 | 输出质量不稳定 | 关键步骤依赖模型即兴发挥 | 将确定性步骤用脚本固化 |
| 4 | 上下文溢出 | 所有内容塞进一个 SKILL.md | 使用渐进式披露，详细内容放入 references |
| 5 | Agent 编造数据 | 指令中未禁止虚构 | 明确写入"严禁编造数据"并设置验证步骤 |
| 6 | 多 Skill 冲突 | description 语义重叠 | 为每个 Skill 设定清晰的边界和区分关键词 |
| 7 | 脚本执行失败 | 路径错误或依赖缺失 | 在指令中写明依赖安装步骤和错误处理方式 |
| 8 | 模板格式错乱 | Agent 不遵循模板 | 在指令中给出正例和反例，并添加格式验证 |
| 9 | Skill 过时失效 | 无版本管理和定期 Review | 建立 Review 机制，记录 changelog |
| 10 | 团队不会用 | 缺少使用文档和培训 | 编写 Quick Start 指南，录制使用演示 |

## 9.2 调试技巧

```bash
# 1. 验证 SKILL.md 的 frontmatter 格式
python -c "
import yaml
with open('SKILL.md') as f:
    content = f.read()
    fm = content.split('---')[1]
    meta = yaml.safe_load(fm)
    assert 'name' in meta, '缺少 name 字段'
    assert 'description' in meta, '缺少 description 字段'
    assert len(meta['description']) > 50, 'description 太短'
    print('✅ Frontmatter 格式正确')
    print(f'   name: {meta[\"name\"]}')
    print(f'   description 长度: {len(meta[\"description\"])} 字符')
"

# 2. 检查目录结构完整性
find . -name "SKILL.md" -exec echo "Found: {}" \;

# 3. 测试 description 的触发效果
# 在 Agent 中尝试以下表述，检查是否正确触发：
# - "帮我写个周报"
# - "生成本周总结"
# - "I need a weekly report"
```

---

# 第十章：总结与行动清单

## 10.1 核心知识图谱

```
                        ┌─── 定义：模块化的 AI 能力单元
                        │
                        ├─── 核心原理：渐进式披露（三层加载）
                        │
              ┌─── 认知 ─┤
              │          ├─── 设计原则：确定性降级 + 流程固化
              │          │
              │          └─── 与 Prompt/MCP/Agent 的关系
              │
              │          ┌─── 四象限筛选法
              ├─── 识别 ──┤
              │          └─── 五步解剖法（录制→抽象→分层→标注→验证）
              │
              │          ┌─── 标准目录（SKILL.md + scripts + references + assets）
   Skill ─────┤          │
   构建       ├─── 结构 ──┤
   全貌       │          ├─── Frontmatter 规范（name + description + metadata）
              │          │
              │          └─── 正文结构（触发条件→前置准备→执行流程→输出规范→异常处理）
              │
              │          ┌─── Tool Wrapper（工具包装器）
              │          ├─── Generator（生成器）
              ├─── 模式 ──┤
              │          ├─── Reviewer（审查器）
              │          ├─── Inversion（反转）
              │          └─── Pipeline（流水线）
              │
              │          ┌─── description 写作技巧
              ├─── 进阶 ──┤── 指令黄金法则
              │          ├─── 脚本调用最佳实践
              │          └─── Skill 组合与编排
              │
              └─── 治理 ──┬─── 生命周期管理
                          ├─── 团队协作模式
                          ├─── 质量度量指标
                          └─── 持续优化循环
```

## 10.2 立即可执行的行动清单

### 🟢 第一周：识别与设计

- [ ] 列出你/团队每周重复做的 Top 10 工作
- [ ] 用四象限法筛选出 3 个最值得沉淀的工作流
- [ ] 选择第 1 个工作流，用五步解剖法完成拆解
- [ ] 确定该工作流适合的设计模式（或组合）

### 🟡 第二周：开发与测试

- [ ] 创建 Skill 目录结构
- [ ] 编写 SKILL.md（先写 description，再写正文）
- [ ] 将确定性步骤编写为 scripts
- [ ] 整理 references 和 assets
- [ ] 设计 3-5 个测试用例并执行测试

### 🔴 第三周：优化与发布

- [ ] 根据测试结果优化指令措辞
- [ ] 测试 description 的触发准确率（至少试 10 种不同表述）
- [ ] 编写使用指南
- [ ] 在团队中推广使用
- [ ] 收集第一轮使用反馈

### 🔵 持续：迭代与扩展

- [ ] 每月 Review 一次 Skill 质量
- [ ] 根据反馈迭代优化
- [ ] 逐步沉淀第 2、3、4 个 Skill
- [ ] 建立团队的 Skill 治理机制

---

## 10.3 最后一句话

> **Skill 的本质不是"写给 AI 的一段话"，而是"将人的经验和智慧编码为机器可执行的标准操作程序"。** 写好一个 Skill，本质上是在做一次知识工程——你在把一位资深员工脑子里的隐性知识，变成可传承、可复用、可进化的数字资产。

这个能力，在 AI 时代，将越来越值钱。



