---
title: "OpenCode搭建Agents专家团"
author: "凉拌糖醋鱼"
date: "2026-06-20 20:49:18"
source: "https://mp.weixin.qq.com/s/3ofZdQgvdi1WOhMATA2R2w"
---

# OpenCode搭建Agents专家团

> 公众号: 凉拌糖醋鱼
> 发布时间: 2026-06-20 20:49:18
> 原文链接: https://mp.weixin.qq.com/s/3ofZdQgvdi1WOhMATA2R2w

---


#

> 关键词：OpenCode、智能体、多Agent协作、AI编程、工作流编排

---

你有没有遇到过这种情况：用 AI 写代码，简单任务还行，一旦需求复杂起来——比如"帮我实现一个带 JWT 和 OAuth 的用户认证系统"——单个 AI 就开始顾此失彼。架构想不清楚，测试写不全，安全漏洞更别指望它自己发现。

问题不在于模型不够聪明，而在于你让一个人干了整个团队的活。

**现实中，一个软件项目需要架构师、前端、后端、测试、安全审计、DevOps……AI开发也是一样。这篇文章就带你看看，如何用 OpenCode 的原生机制，零代码搭建一个 29 人的 AI 智能体专家团。**

## 一、OpenCode 的智能体机制：.md 文件即 Agent

在开始之前，先搞清楚一个核心概念：**OpenCode 的智能体配置，本质上就是一个 .md 文件。**

不需要写代码、不需要搭框架、不需要启动服务。一个 YAML 头部定义元信息，一段 Markdown 正文定义系统提示词，就是一个完整的智能体。

### 1.1 配置文件结构

每个智能体文件的格式如下：


```sql
---
description: 架构设计智能体。负责系统级设计和技术方案规划
mode: subagent
model: opencode-go/glm-5.1
temperature: 0.4
tools:
 write: false
 edit: false
 bash: false
---

You are a senior software architect who designs scalable,
maintainable, and robust systems...
```


各字段含义：

| 字段 | 作用 | 说明 |
| --- | --- | --- |
| `description` | 触发描述 | OpenCode 根据这段中文描述决定何时调用该智能体 |
| `mode` | 运行模式 | `primary`（主智能体，可独立会话）或 `subagent`（子智能体，被调度调用） |
| `model` | 使用的模型 | 格式为 `provider/model-name`，不同智能体可用不同模型 |
| `temperature` | 创造性参数 | 0.0-1.0，架构师偏保守（0.4），代码生成偏稳定（0.3） |
| `tools` | 工具权限 | 控制该智能体能否写文件、执行命令等 |

**关键洞察：`description` 字段必须写中文。** 这是 OpenCode 用来匹配用户请求和智能体的触发条件。写得好不好，直接决定智能体能不能被正确调度。

### 1.2 主智能体 vs 子智能体

OpenCode 的智能体分两级：

- **主智能体（Primary）**：可以作为顶层会话直接与用户对话，相当于"项目经理"
- **子智能体（Subagent）**：只能被主智能体通过 `@agent-name` 方式调度，相当于"专项负责人"

在 opencode-agentcrew 项目中，设了两个主智能体：

| 主智能体 | 模型 | 定位 |
| --- | --- | --- |
| **Erribaba** | mimo-v2.5-pro | 生产代码、复杂算法、深度分析——"认真的那个" |
| **Zero** | mimo-v2.5 | 快速原型、多模态输入、轻量任务——"灵活的那个" |

选哪个当默认主智能体，取决于你的工作风格。写正式项目用 Erribaba，快速验证想法用 Zero。

## 二、29 个专家怎么分工？

这是整个方案最核心的部分。opencode-agentcrew 把软件开发流程拆成了 29 个角色，每个角色对应一个 .md 文件。

### 2.1 角色矩阵

按职能分为 6 大类：

- **工作流与编排（3 个）**

- `workflow-orchestrator` — 工作流编排，管理完整的 开发生命周期
- `plan-writer` — 实现计划编写，将需求分解为 TDD 任务
- `project-manager` — 需求分析、任务拆解、Sprint 规划

- **代码质量与审查（5 个）**

- `reviewer` — 代码质量审查（Stage 2）
- `spec-reviewer` — 规范合规性审查（Stage 1）
- `frontend-reviewer` — 前端代码审查
- `security-auditor` — OWASP Top 10 安全审计
- `validator` — 最终验证（构建/测试/类型检查）

- **代码实现（6 个）**

- `code-generator` — 高质量代码生成
- `software-engineer` — 全栈功能端到端实现
- `frontend-dev` — React/Vue/Svelte 组件开发
- `ui-designer` — CSS/Tailwind/响应式布局
- `refactorer` — 代码重构
- `vision-dev` — 设计稿分析、截图还原

- **测试（2 个）**

- `test-writer` — 单元/集成/边界测试
- `e2e-tester` — Playwright/Cypress 端到端测试

- **基础设施（5 个）**

- `architect` — 系统架构设计
- `api-designer` — API 端点设计、OpenAPI 规范
- `db-engineer` — 数据库 Schema、Migration、SQL 优化
- `devops` — Docker、CI/CD、Kubernetes
- `migration` — 框架升级、数据库迁移

- **辅助（8 个）**

- `debugger` — 系统化定位和修复缺陷
- `perf-optimizer` — 性能分析与优化
- `executor` — 运行命令、执行测试
- `research` — 查找文档、调研技术方案
- `doc-writer` — 技术文档、API 参考
- `git-assistant` — 提交消息、分支命名、PR 描述

### 2.2 权限控制：谁能写、谁能执行

不是所有智能体都需要同样的权限。opencode-agentcrew 通过 `tools` 字段做了精细控制：

| 权限类型 | 说明 | 示例 |
| --- | --- | --- |
| 只读 | `write:false, edit:false, bash: false` | architect、reviewer、research |
| 可写 | `write:true, edit: true` | code-generator、plan-writer |
| 可执行 | `write:true, edit:true, bash: true` | debugger、devops、executor |

**为什么要区分？** 因为架构师只需要"看"和"说"，不应该有权限直接改代码；而 debugger 需要跑命令来复现问题，必须有 bash 权限。权限最小化原则，跟真实团队一模一样。

### 2.3 多模型协作：不同角色用不同模型

这是opencode最聪明的设计之一——**不同智能体可以使用不同的模型**。

| 角色 | 模型 | 为什么选它 |
| --- | --- | --- |
| 架构师 | GLM-5.1 | 系统设计需要强推理能力 |
| 代码生成 | mimo-v2.5-pro | 代码生成质量高 |
| 调试诊断 | DeepSeek V4 Pro | 长上下文分析能力强 |
| 前端开发 | Kimi K2.6 | 前端框架理解好 |
| 文档编写 | Qwen3.7 Plus | 中文表达流畅 |
| 命令执行 | MiniMax M2.7 | 工具调用稳定 |

**核心思想：没有一个模型是全能的，但每个模型都有擅长的领域。把对的模型用在对的地方，比用一个"最强模型"干所有事效果更好。**

## 三、结构化工作流：从混乱到有序

有了 29 个专家，下一个问题是：**谁先干、谁后干、怎么配合？**

opencode-agentcrew引入了一套结构化工作流，参考了MiMo Compose 模式，把开发流程分为 5 个阶段：


```css
用户需求
 ↓
[阶段 1: 构思] — architect + research
 ↓
[阶段 2: 计划] — plan-writer + project-manager
 ↓
[阶段 3: 执行] — test-writer → code-generator → executor
 ↓
[阶段 4: 审查] — spec-reviewer (Stage 1) → reviewer (Stage 2)
 ↓
[阶段 5: 合并] — validator + git-assistant
```


### 3.1 两阶段审查：最关键的创新

传统AI写完代码就交差了，没人检查写得对不对。opencode-agentcrew 设计了**两阶段审查机制**：

**Stage 1：规范合规性审查（spec-reviewer）**

- 问题：实现是否符合原始需求？
- 方法：用规范锚定（[Sn] 引用），逐条对照需求验证
- 证据优先：每个结论必须有代码证据支撑

**Stage 2：代码质量审查（reviewer）**

- 问题：代码是否写得好？
- 检查项：逻辑错误、安全漏洞、性能问题、命名规范、类型安全、错误处理
- **前提：Stage 1 必须通过，Stage 2 才会运行**

**为什么分两步？**因为"写对了"和"写好了"是两件事。先确认方向没错，再打磨细节——这跟真实团队的 Code Review 流程完全一致。

### 3.2 TDD 集成：先写测试再写代码

执行阶段强制遵循测试驱动开发（TDD）：


```css
RED
：test-writer 编写失败测试
VERIFY
：executor 确认测试确实失败
GREEN
：code-generator 写最小实现让测试通过
VERIFY
：executor 确认测试通过
REFACTOR
：如需要，refactorer 重构代码
COMMIT
：git-assistant 提交
```


**这解决了一个大问题：AI 写的代码"看起来对"但实际有 bug。** 有了测试先行，至少能保证功能层面的正确性。

### 3.3 并行执行：background=true

独立的任务可以并行执行。关键配置：


```bash
 # shell 配置中必须开启
export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
```


使用时：


```typescript
// ❌ 同步阻塞，串行执行
@code-generator (任务A)
@ui-designer (任务B)

// ✅ 异步并行，同时执行
@code-generator (任务A, background=true)
@ui-designer (任务B, background=true)
```


但并行不是无脑的——**阶段屏障（Phase Barrier）** 确保"构思"全部完成才能进入"计划"，"审查"全部通过才能进入"合并"。该快的快，该等的等。

## 四、实战示例：一个需求的完整旅程

来看一个具体例子。假设用户说：

> "帮我实现一个用户认证系统，支持 JWT 和 OAuth"

workflow-orchestrator 会这样调度：

### 阶段 1：构思


```typescript
@architect → 设计认证架构（JWT + OAuth 的服务层抽象）
@research → 调研 JWT 和 OAuth 最佳实践
```


输出：架构方案 + 技术选型决策

### 阶段 2：计划


```css
@plan-writer → 创建实现计划：
 - Task 1: 编写 JWT 中间件测试 (TDD)
 - Task 2: 实现 JWT 验证
 - Task 3: 编写 OAuth 集成测试 (TDD)
 - Task 4: 实现 OAuth 流程
 - Task 5: 集成测试
 - Task 6: 文档更新
```


### 阶段 3：执行


```typescript
@test-writer → 编写 JWT 中间件的失败测试
@executor → 确认测试失败
@code-generator → 实现 JWT 中间件
@executor → 确认测试通过
...（循环直到所有任务完成）
```


### 阶段 4：审查


```typescript
@spec-reviewer → 验证：JWT 验证逻辑是否符合需求？OAuth 流程是否完整？
（Stage 1 通过后）
@reviewer → 检查：代码质量、安全漏洞、错误处理
@security-auditor → 审计：OWASP Top 10 检查
```


### 阶段 5：合并


```typescript
@validator → 最终验证：构建通过、测试全绿、类型检查无误
@git-assistant → 生成规范的提交消息并提交
```


**整个过程，用户只需要说一句话。** 后面的 29 个专家自动配合，像一个真正的开发团队在运转。

## 五、3 分钟上手

## 5.1 一键安装

**Linux / macOS：**


```css
用户需求
 ↓
[阶段 1: 构思] — architect + research
 ↓
[阶段 2: 计划] — plan-writer + project-manager
 ↓
[阶段 3: 执行] — test-writer → code-generator → executor
 ↓
[阶段 4: 审查] — spec-reviewer (Stage 1) → reviewer (Stage 2)
 ↓
[阶段 5: 合并] — validator + git-assistant
```
0

**Windows (PowerShell)：**


```css
用户需求
 ↓
[阶段 1: 构思] — architect + research
 ↓
[阶段 2: 计划] — plan-writer + project-manager
 ↓
[阶段 3: 执行] — test-writer → code-generator → executor
 ↓
[阶段 4: 审查] — spec-reviewer (Stage 1) → reviewer (Stage 2)
 ↓
[阶段 5: 合并] — validator + git-assistant
```
1

脚本会自动下载 29 个智能体文件到 `~/.config/opencode/agents/`，并备份已有配置。

### 5.2 手动安装


```css
用户需求
 ↓
[阶段 1: 构思] — architect + research
 ↓
[阶段 2: 计划] — plan-writer + project-manager
 ↓
[阶段 3: 执行] — test-writer → code-generator → executor
 ↓
[阶段 4: 审查] — spec-reviewer (Stage 1) → reviewer (Stage 2)
 ↓
[阶段 5: 合并] — validator + git-assistant
```
2

### 5.3 配置模型

安装后需要为每个智能体配置你可用的模型。最简单的方式是把 README 中的配置提示词发给 OpenCode，它会自动完成：

1. 读取你的 `opencode.json`，列出已有 provider 和模型
2. 为每个智能体匹配可用模型
3. 更新所有 .md 文件中的 `model` 字段
4. 设置默认主智能体

### 5.4 开始使用


```css
用户需求
 ↓
[阶段 1: 构思] — architect + research
 ↓
[阶段 2: 计划] — plan-writer + project-manager
 ↓
[阶段 3: 执行] — test-writer → code-generator → executor
 ↓
[阶段 4: 审查] — spec-reviewer (Stage 1) → reviewer (Stage 2)
 ↓
[阶段 5: 合并] — validator + git-assistant
```
3

## 六、这套方案的本质：用配置替代代码

回头看整个 opencode-agentcrew 项目，它做的事情其实很简单：

**没有一行代码。** 29 个 .md 文件 + 1 个安装脚本，就是一个完整的多智能体开发系统。

这揭示了OpenCode 智能体系统的一个重要特性：**它是面向配置的，不是面向编程的。** 你不需要会写代码，就能搭建一个复杂的AI专家团队。你需要的是：

1. **理解角色分工** — 谁该干什么
2. **写好 description** — 让 OpenCode 知道什么时候调用谁
3. **选对模型** — 让擅长的模型做擅长的事
4. **设计工作流** — 让协作有章法

这四点，比写代码重要得多。

---

**一个人用AI写代码，是工具；一群人用AI协作，才是生产力。**OpenCode 的智能体机制，本质上是把"一群人协作"的组织逻辑，变成了可配置、可复用、可分享的 .md 文件。你搭好的不只是一套开发工具，更是一个可以不断进化的 AI 团队。

---

> 项目地址：https://gitee.com/aerlee/opencode-agentcrew
> 许可证：MIT
> 推荐配合：OpenCode Go 套餐（一次订阅，12 个模型可用）

