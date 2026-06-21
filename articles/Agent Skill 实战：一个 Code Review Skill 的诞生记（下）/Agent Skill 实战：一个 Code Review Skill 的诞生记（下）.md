---
title: "Agent Skill 实战：一个 Code Review Skill 的诞生记（下）"
author: "昀启AI+"
date: "2026-03-26 07:09:57"
source: "https://mp.weixin.qq.com/s/Abd8aP8QRxUzVtr9ZTZLCg"
---

# Agent Skill 实战：一个 Code Review Skill 的诞生记（下）

> 公众号: 昀启AI+
> 发布时间: 2026-03-26 07:09:57
> 原文链接: https://mp.weixin.qq.com/s/Abd8aP8QRxUzVtr9ZTZLCg

---
![Image](images/img_001.png)上篇《[Agent Skill 实战：一个Code Review Skill 的诞生记（上）](https://mp.weixin.qq.com/s?__biz=MzU5NDY2NzQyNw==&mid=2247483793&idx=1&sn=7e14c7688bc548a0836c9c0849ad66b4&scene=21#wechat_redirect)》把一个 Code Review Skill 的骨架拆开了：`SKILL.md` 负责流程引擎，`reference.md` 负责项目速查，`.mdc` 规则文件负责逐层规范。

在有了骨架之后，还有几件事也是需要做的：

- 把通用骨架变成你团队自己的检查标准
- 理解 Token 消耗，避免“裸 Prompt”的不稳定且烧预算
- 从 `code-review` 推广到其他 Skill

## 本地化：怎么让这个 Skill 变成你的

##

前面展示的 `code-review` Skill 是一个已经半本地化过的例子。它绑定了 COLA 架构和团队自己的 Java 编码规范。你要做的不是照抄检查项，而是用同一个推导公式，从你自己团队的约束里推导出你的检查项：


```
你的架构/编码约束 → 团队实际踩过的坑 → Agent 可执行的检查标准
```


本地化要绑定三个东西：架构约定、编码规范、上下文路由。前两者决定“检查什么”，后者决定“运行时加载什么”。

### 1. 架构检查项的推导

以 COLA 为例，核心依赖规则是：

```
adapter → application → domain ← infrastructure
```

从这条规则出发，再结合团队真正踩过的坑，就能推导出可以让 Agent 直接执行的检查项。

| 检查项 | 架构约束 | 团队踩过的坑 | Agent 检查标准 |
| --- | --- | --- | --- |
| 依赖方向 | 四层单向依赖 | Service import 了 infrastructure 实现类，人工 CR 容易漏 | 扫描各层 import 声明，检查反向依赖 |
| Gateway 抽象 | Service 通过 Gateway 接口访问数据 | 直接注入 Mapper 导致单测必须启动 MyBatis 上下文，解耦不够好，后续换存储改动面极大 | Service 中是否存在直接注入 Mapper 实现类 |
| domain 零依赖 | domain 只有纯 Java | Entity 加 `@Data`、Domain Service 加 `@Service`、Gateway 参数用了 DO | domain 包下是否引入 `org.springframework.*` 等框架包 |
| Entity 使用时机 | Entity 承载业务行为 | 纯查询也建 Entity；有状态流转却在 Service 里写 if-else | 写操作是否通过 Entity 方法；纯查询是否误建 Entity |
| 层间数据传递 | 各层用各自数据对象 | Request 透传到 domain，DO 直接暴露到接口返回值 | 各层方法签名是否使用正确层级的数据对象 |

这一步最关键的，不是列出“原则”，而是把原则翻译成**可执行的判断信号**。只有这样，Agent 才不是“懂一点道理”，而是真的“会检查”。

### 2. 编码规范检查项的推导

编码规范也是同样的逻辑。不要只写“遵守最佳实践”，而要写成“什么情况算违规”。

| 检查项 | 为什么这么规定 | Agent 检查标准 |
| --- | --- | --- |
| 禁止 `@Autowired` 字段注入 | 隐藏依赖数量、无法 final、脱离容器无法测试 | 类中是否存在 `@Autowired` / `@Resource` 标注在字段上 |
| `@Transactional` 必须声明 `rollbackFor` | Spring 默认不回滚 checked Exception，会导致数据不一致的问题 | 写操作的 `@Transactional` 是否包含 `rollbackFor = Exception.class` |
| MapStruct `id` / 审计字段 ignore | 自动映射 id 可能覆盖主键；审计字段容易被误覆盖 | Convertor 中 `id/createTime/updateTime` 等是否标注 ignore |
| 命名规范 | 命名不统一会让代码风格迅速失控 | 是否符合类名 PascalCase、DTO 后缀全大写、URL kebab-case、JSON snake\_case |
| 异常处理 | 入口层和业务层的异常职责不同 | Controller 是否有不必要的 try-catch；Service 异常是否有业务语义 |
| 日志规范 | 关键操作要能追，敏感信息不能漏 | 是否补齐关键日志；是否打印敏感数据 |

###

### 3. 业务逻辑实现方式的检查

###

架构和编码规范回答的是“结构对不对”和“写法好不好”。但代码评审还有一个很容易被忽视的维度：**核心业务逻辑的实现方式是否合理**。

这部分通常不写在 `SKILL.md` 的固定维度里，而是通过 `.mdc` 规则文件按业务模块补充。

比如在交易场景下，`application-service.mdc` 可能会包含这样的业务实现约束：

| 检查关注点 | 具体要求 |
| --- | --- |
| 金额计算 | 金额字段必须使用 `BigDecimal`，禁止 `double/float`；运算必须指定 `RoundingMode` |
| 幂等控制 | 涉及资金变动的写接口必须实现幂等（基于业务唯一键或幂等 Token） |
| 状态流转 | 业务状态变更必须通过 Entity 方法实现，禁止直接 set 状态字段；状态机需做前置校验，拒绝非法流转 |
| 多租户隔离 | 涉及数据查询的 SQL 必须包含 `tenantId` 条件，禁止无租户限定的全表查询 |
| 并发控制 | 对共享资源的修改操作需要考虑并发场景，视情况使用乐观锁（版本号）或分布式锁 |
| 外部系统调用 | 调用外部 API 必须有超时设置、重试策略和降级方案；禁止在事务内调用外部服务 |

这类检查项不写在 `SKILL.md` 里，是因为它们和具体业务强相关；但它们又确实是你团队最容易踩坑的地方。所以最合适的归宿，就是对应模块的 `.mdc` 规则文件。

### 4. 单测要求与质量门禁

`SKILL.md` 的维度 7（测试充分性）定义了通用测试检查项。但在实际落地中，团队通常还会在 `.mdc` 规则文件中补充更具体的单测要求和 CI/CD 质量门禁标准。

**单测要求：**

- 核心业务逻辑的 Service 方法必须有对应的单元测试
- 测试方法命名遵循 `should_预期行为_when_前置条件` 格式
- 每个测试方法只测一个行为，禁止一个 `@Test` 里塞多个断言场景
- Mock 外部依赖（Gateway、外部 API），不 Mock 被测试类本身的方法
- 使用 AssertJ 链式断言，禁止只 assert 不抛异常

**质量门禁：**

- 核心模块单测行覆盖率不低于 70%，新增代码覆盖率不低于 80%
- SonarQube 扫描无 Blocker/Critical 级别问题
- 变更代码必须通过 `code-review` Skill 评审，且无未解决的 Critical 项

这些标准写在 `java-coding-style.mdc` 或各层 `.mdc` 文件中，Agent 在评审时会参照执行。

### 5. 上下文路由：三个文件怎么配合加载

###

Agent 在 Step 1 确定评审范围后，会按以下逻辑加载上下文：


```bash
Step 1 加载顺序：
    │
    ├── ① 读取 reference.md（项目级速查手册，每次都加载）
    │      → 违规模式速查、高关注区域、测试门禁路径
    │
    ├── ② 按代码包路径，加载对应层的 .mdc 规则文件
    │      adapter/web/    → adapter-web.mdc
    │      application/    → application-service.mdc
    │      domain/         → domain-layer.mdc
    │      infrastructure/ → infrastructure-impl.mdc
    │      **/convertor/** → mapstruct-convertor.mdc
    │      src/test/**     → unit-test.mdc
    │
    └── ③ 加载通用规则（每次都加载）
           cola-architecture.mdc + java-coding-style.mdc
```


`reference.md` 是项目级的，体积相对小，所以每次评审都加载。

`.mdc` 规则文件是按需加载的。评审 Controller，不会加载 `domain-layer.mdc`；评审 Entity，也不会加载 `adapter-web.mdc`。

这就是为什么我们前面一直强调：**结构化 Skill 的重点，不只是“提示词怎么写”，而是“上下文怎么路由”。**

### 6. 本地化步骤总结

###

如果你的项目还没有这套结构，本地化的落地顺序可以直接照着来：

```
Step 1: 从架构约束出发，按“约束 → 踩坑 → 可执行标准”推导架构检查项
Step 2: 从编码约定出发，用同样方法推导编码检查项
Step 3: 梳理核心业务逻辑的实现约束，写入对应模块的 .mdc 规则文件
Step 4: 明确单测要求和质量门禁标准，写入unit-test.mdc/java-coding-style.mdc
Step 5: 将规范按架构层拆成独立 .mdc 规则文件，建立路由表
Step 6: 整理项目特有的业务模式和违规速查，写入 reference.md
Step 7: 用真实代码跑几次评审，校准检查项粒度和严重性分级
```

```

```

## 一次评审到底消耗多少Token

##

我们在探索过程中，也曾走捷径直接用一段裸 Prompt 让 Agent干活，扮演 Coder、Reviewer、Tester 协同工作。结果就是它用自己的理解，代码中真实存在的问题查不出来。输出时而查架构时而查命名，格式一会儿列表一会儿散文，而且因为没有上下文路由，每次都把所有能读的文件全读一遍。两周下来，预算烧完了，输出质量就像在抽奖。这就是没有结构化 Skill 定义的代价。

回到 Sub-agent 的架构下，Token 消耗仍然是一个绕不开的问题。但和裸 Prompt 不同的是，结构化之后消耗变得**可预测、可优化**。不需要精确到个位数，但需要理解消耗的结构，才能做有效的成本控制。

### 消耗在哪里

###

Agent 执行一次代码评审，Token 消耗分两大块。

**Input Token（占 75%-85%）**：你“喂”给 Agent 的上下文

- `SKILL.md` 本身（约 3,500 Token）
- `reference.md` 速查手册（约 1,500-2,000 Token，每次都加载）
- 待评审代码（按文件数和行数线性增长，3-5 个文件约 5K-10K Token）
- 加载的 `.mdc` 规则文件（2-3 个约 3K-6K Token，按需加载）
- Agent 自动读取的模块上下文（约 2K-4K Token）
- 系统 Prompt 和对话历史（约 2K-2.5K Token）

**Output Token（占 15%-25%）**：Agent 输出的评审报告

- 结构化报告含问题描述、修复建议、代码示例（约 3K-5K Token）

一次中等规模的评审（3-5 个文件，500-800 行代码），总消耗大约在 **20K-34K Token** 之间。

### 2. 花多少钱

###

用主流模型跑一次，成本大约在 **0.1-0.15 美元**，折合人民币不到一块钱。按团队每天 10 次评审（看实际团队规模）、每月 22 个工作日算，月度总成本大约 **150-230 元**。

对比一下：一位高级工程师每天花在 CR 上 30-60 分钟，时间折算下来远超这个数字。Agent 做不到完全替代人工 CR，但能把大量机械性检查（依赖方向、命名规范、事务注解、敏感数据暴露）先过一遍，让人把精力放在架构决策和业务逻辑判断上。

### 3. 成本优化的关键

###

从拆解可以看出一个重要结论：**Token 消耗的大头不是 Skill 本身，而是它需要加载的上下文。**

`SKILL.md + reference.md` 合计只占总消耗的 15%-22%，真正的大头是待评审代码和 `.mdc` 规则文件。这意味着：

1. **不要为了“省 Token”简化检查项**。检查项不够具体，Agent 输出质量下降，反而会拉高总成本。
2. **上下文路由是最高 ROI 的优化**。按代码所在层只加载对应的 `.mdc` 规则文件，而不是全量加载。
3. **`reference.md` 保持精简**。它每次都加载，所以只放速查表和模式速查，不要把详细规范也塞进去。
4. **缩小评审范围也很有效**。日常评审只看 Git diff 变更部分，而不是整个模块

## 从 Code Review 推广到其他 Skill

##

推广到其他 Skill 时，骨架完全一样，差异在内容。用 `write-prd` 做一个对照：

| 结构层 | code-review | write-prd |
| --- | --- | --- |
| Frontmatter | 触发词：“CR”“review”“代码评审” | 触发词：“PRD”“需求文档”“用户故事” |
| Step 1 | 确定评审范围 + 加载上下文 | 需求澄清（5 个必问问题） |
| Step 2 | 逐维度检查（7 个维度） | 需求拆解（角色/功能/模糊点） |
| Step 3 | 生成评审报告 | 生成用户故事 + PRD 文档 |
| Step 4 | 输出 + 保存 | 自检 checklist + 输出确认 |
| 原则 | 对事不对人、项目约定优先 | toB 视角优先、语言精确、不过度设计 |

核心设计原则是一样的：**把一类工作拆成明确步骤，每一步定义清楚输入、执行标准和输出格式。**

## 写 Skill 最容易犯的错误

##

最后分享几个最常见的坑：

**1. 检查项只有名字，没有违反信号。**“检查依赖注入方式”这种写法，对 Agent 来说几乎等于没写。“类中是否存在 `@Autowired` 字段注入”，它才知道到底要查什么。

**2. 把所有内容都塞进 `SKILL.md`。**`SKILL.md` 只管流程和维度；逐层编码规范放 `.mdc`；项目特有的违规模式和业务领域知识放 `reference.md`。全塞进一个文件的后果，就是文件过大、无法按需路由、规范更新时牵一发而动全身。

**3. 忘了写原则。**步骤不可能覆盖所有情况。没有“区分严重性”原则，Agent 很容易把所有疑似问题都标成 Critical。

**4. 没有输出模板。**没有模板，Agent 每次输出格式都不同，团队很难形成稳定的阅读习惯。

**5. 第一版就想覆盖所有维度。**更稳妥的做法是先跑 3-4 个核心维度验证效果，再逐步扩展。第一版一口气铺满，往往会把自己先校准崩。

## 总 结

有人可能会问，为什么选 Code Review 做案例？因为它是所有研发团队的日常刚需，输入输出明确（代码进、分级报告出），天然需要绑定团队规范，维度又足够丰富。一个 Skill，就能把从输入到检查再到输出的全链路跑通。

这篇文章拆解下来，核心其实就三件事：

**第一，Skill 的骨架是通用的。** 任何 Skill 都要回答四个问题：什么时候调用、做到什么程度、按什么步骤做、不确定时怎么判断。

**第二，一个 Skill 不是一个文件，而是三个文件协作。**`SKILL.md` 定义流程和维度，`reference.md` 提供项目特有的违规模式和业务领域知识速查，`.mdc` 规则文件定义每一层的具体编码规范。三者各司其职，按需加载。

**第三，本地化不是填模板，而是一次推导。** 推导公式始终是：

```
架构/编码/业务约束 → 团队实际踩过的坑 → Agent 可执行的检查标准
```

```

```

如果你的团队正准备写第一个 Skill，建议从 `code-review` 开始。不只是因为它实用，更因为本地化的过程会倒逼你做一件早就该做的事：把团队的架构约定和编码规范从“口口相传”变成“白纸黑字”。这些沉淀下来的规范文件，不管 Agent 用不用，本身就已经是一笔值得长期维护的团队资产。

---

*上篇回看：《[Agent Skill 实战：一个Code Review Skill 的诞生记（上）](https://mp.weixin.qq.com/s?__biz=MzU5NDY2NzQyNw==&mid=2247483793&idx=1&sn=7e14c7688bc548a0836c9c0849ad66b4&scene=21#wechat_redirect)》*


