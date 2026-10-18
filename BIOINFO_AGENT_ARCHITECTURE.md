# 生物信息科研知识问答 Agent 助手

**项目角色：** 核心成员　 **时间：** 2026.03 - 2026.06  
**技术栈：** Python、FastAPI、Claude、DeepSeek Compatible API、Redis、ChromaDB、Docker

## 项目简介

面向生物信息科研场景的 LLM 知识问答与任务执行平台。系统通过细粒度意图识别和结构化主辅 Agent 路由处理知识问答、miRNA-drug 关联预测及复杂研究请求；按需检索领域知识、注入 Skills，并通过 MCP 工具执行预测、网页信息获取和测试报告生成。平台使用 Redis 与 ChromaDB 管理会话和长期语义记忆，并结合安全沙箱、链路 Hook 和 LLM-as-Judge 评测，提升执行过程的可控性与结果质量。

## 系统总览

```mermaid
flowchart LR
    U[科研用户] --> API[FastAPI 接入层]
    API --> G[请求治理与链路 Hook<br/>权限校验 / 缓存 / 重试 / 审计]
    G --> MEM[记忆上下文装配]
    MEM <--> R[(Redis<br/>会话工作记忆 / 计划 / 中间结果)]
    MEM <--> C[(ChromaDB<br/>历史摘要 / 研究偏好 / 知识库)]
    MEM --> I[细粒度意图识别<br/>Few-shot LLM + Embedding + Pattern]
    I -->|知识问答| A[主 Agent / Orchestrator]
    I -->|关联预测| A
    I -->|复杂研究任务| A
    I -->|需人工处理| H[转人工 / 澄清]

    A --> P[任务规划与状态管理]
    P --> S[动态 Skills 注入]
    P --> SUB[辅 Agent / 专项执行器]
    SUB -->|知识增强| Q[RAG 检索管线]
    Q --> C
    Q -->|并行子查询 / 去重 / 重排| SUB
    SUB -->|工具请求| MCP[MCP 工具路由]
    MCP --> T1[miRNA-drug 预测与关联分数]
    MCP --> T2[fetch-web 信息获取]
    MCP --> T3[测试报告生成]
    S --> SUB
    SUB --> BOX[OpenSandbox 主机-沙箱双层访问平面]
    BOX -->|隔离网络 / 进程 / 文件系统| EXEC[代码脚本与高风险 Skill 执行]
    T1 --> V[结果校验与证据汇总]
    T2 --> V
    T3 --> V
    Q --> V
    V -->|状态更新| P
    V --> RESP[回答 / 结构化结果 / 报告]
    RESP --> API
    RESP --> J[LLM-as-Judge 端到端评测]
    G -.模型 / 工具 / 文件 / 状态事件.-> TRACE[链路追踪、敏感操作拦截与审计]
    P -.任务状态事件.-> TRACE
    MCP -.工具事件.-> TRACE
    BOX -.文件与执行事件.-> TRACE
```

### 架构职责

| 层级 | 核心职责 |
| --- | --- |
| 接入与安全层 | 接收用户请求，完成权限检查、敏感操作识别和审计准备 |
| 意图与路由层 | 识别用户意图、抽取实体，选择问答、预测、转人工等处理路径 |
| Agent 编排层 | 拆解任务、维护依赖、调度工具与知识检索，并汇总中间结果 |
| 知识增强层 | 对复杂查询进行拆分、改写、召回、去重和 LLM 重排 |
| 工具执行层 | 通过 MCP 调用 miRNA-drug 预测、网页信息获取和测试报告生成能力 |
| 记忆层 | 使用 Redis 管理短期工作状态，使用 ChromaDB 管理长期语义记忆和知识 |
| 执行隔离层 | 对代码脚本、文件访问和 Skill 调用进行网络、进程及文件系统隔离 |
| 观测评测层 | 在关键节点注入 Hook，并通过 LLM-as-Judge 进行端到端评测 |

## Agent 详细执行流程

```mermaid
flowchart TD
    START([收到用户请求]) --> LOAD[读取会话工作记忆与历史偏好]
    LOAD --> CLASSIFY[意图识别与实体抽取<br/>语义 Few-shot / Embedding / Pattern 融合]
    CLASSIFY --> ROUTE{路由决策}
    ROUTE -->|缺少关键实体| CLARIFY[生成澄清问题]
    ROUTE -->|知识问答| PLAN
    ROUTE -->|miRNA-drug 预测| PLAN
    ROUTE -->|复合研究任务| PLAN
    ROUTE -->|需人工处理| HUMAN[转人工]

    PLAN[主 Agent 制定依赖任务清单] --> STATE[保存任务与状态<br/>pending / in_progress / completed / failed]
    STATE --> NEXT[选取依赖已满足的待执行子任务]
    NEXT --> SKILL[按任务动态注入 Skills]
    SKILL --> KIND{子任务类型}
    KIND -->|领域知识| RAG
    KIND -->|预测 / 查询| TOOL
    KIND -->|网页信息| WEB
    KIND -->|报告生成| REPORT

    RAG[构造多角度子查询] --> PAR[并行检索 ChromaDB]
    PAR --> DEDUP[内容哈希与相似度去重]
    DEDUP --> RERANK[LLM 重排与证据整理]
    TOOL[调用 MCP 预测工具] --> TOOLCHK[校验参数与返回结构]
    WEB[调用 MCP fetch-web] --> WEBCHK[整理来源与相关信息]
    REPORT[调用 MCP 报告工具] --> SANDBOX
    SKILL -.高风险 Skill / 脚本.-> SANDBOX[OpenSandbox 隔离执行]
    SANDBOX --> SANDBOXCHK[校验文件访问与执行结果]

    RERANK --> RESULT[子任务结果与证据]
    TOOLCHK --> RESULT
    WEBCHK --> RESULT
    SANDBOXCHK --> RESULT
    RESULT --> SAVE[写入 Redis 中间结果并更新任务状态]
    SAVE --> MORE{还有可执行子任务?}
    MORE -->|有| NEXT
    MORE -->|无| VALIDATE[主 Agent 汇总并校验完整性 / 一致性]
    VALIDATE -->|证据不足或任务失败| RECOVER[有限重试 / 调整计划 / 请求澄清]
    RECOVER --> STATE
    VALIDATE -->|通过| ANSWER[生成面向用户的回答或结构化报告]
    ANSWER --> MEMORY[更新会话工作记忆；提炼摘要与研究偏好]
    MEMORY --> EVAL[LLM-as-Judge 评估相关性 / 准确性 / 完整性 / 有用性]
    EVAL --> END([返回结果并记录评测与审计事件])

    TRACE[链路 Hook：模型调用 / 工具调用 / 文件读写 / 状态变更] -.观测与治理.-> CLASSIFY
    TRACE -.观测与治理.-> TOOL
    TRACE -.观测与治理.-> SANDBOX
    TRACE -.观测与治理.-> SAVE
```

## 核心模块

### 1. 细粒度意图识别

融合 LLM Few-shot 语义理解、Embedding 相似度和 Pattern 关键词匹配，对请求进行意图分类与实体抽取，覆盖知识问答、关联预测、复合任务及转人工等场景。识别结果作为 Agent 路由和任务规划的输入。

### 2. 主辅 Agent 与任务规划

主 Agent（Orchestrator）负责理解请求、规划步骤、管理依赖、汇总结果和处理失败；辅 Agent 或专项执行器负责知识检索、工具调用、信息整理和报告生成。任务清单记录依赖关系及 `pending`、`in_progress`、`completed`、`failed` 状态，支持中间结果持久化和按状态恢复。

典型复合任务链路：

```text
请求解析 -> MCP 工具调用 -> 预测结果校验 -> RAG 查询基础机制 -> 报告生成 -> 结果汇总
```

### 3. 意图触发的 RAG

仅在需要领域知识增强时触发检索。对于复杂问题，先拆分并改写为多角度子查询，再并行召回 ChromaDB 文档；随后结合内容哈希与相似度去重，并由 LLM 重排相关结果，形成供 Agent 使用的证据上下文。

### 4. MCP 工具与动态 Skills

- **miRNA-drug 工具：** 输入药物获取 Top-N miRNA 候选，或查询指定药物-miRNA 组合的关联分数。
- **fetch-web：** 获取外部网页信息，供研究问题补充参考。
- **测试报告生成：** 根据任务结果生成结构化报告。
- **动态 Skills：** 根据任务类型按需加载领域流程与提示能力，避免无关上下文进入当前任务。

### 5. 分层记忆

- **Redis 会话工作记忆：** 异步保存当前会话上下文、任务计划、状态和中间结果。
- **ChromaDB 长期语义记忆：** 保存历史对话摘要、用户研究偏好及领域知识文档，支持相似度检索。
- **非阻塞访问：** 对 ChromaDB 等同步向量库操作放入线程池执行，减少对异步请求主链路的阻塞。

### 6. 安全沙箱与链路治理

基于 OpenSandbox 构建主机与沙箱的双层文件访问平面，对网络、进程和文件系统进行隔离；代码脚本执行及高风险 Skill 调用在沙箱内运行。模型调用、工具调用、文件读写和任务状态变更等节点注入 Hook，集中处理缓存、重试、权限校验、敏感操作拦截和审计上报。

### 7. 端到端评测

通过 LLM-as-Judge 对最终回复进行相关性、准确性、完整性和有用性评估，并可用于意图识别及端到端回归检查，为提示词、检索策略和 Agent 执行流程迭代提供反馈。

## 典型请求示例

```text
用户：查询药物 X 关联度最高的 Top-10 miRNA，并说明可能的调控机制。

1. 意图识别：关联预测 + 机制知识问答
2. 实体抽取：药物 X、Top-10、调控机制
3. 任务规划：预测 -> 结果校验 -> 机制 RAG -> 汇总报告
4. MCP 调用：返回 Top-10 miRNA 及关联分数
5. RAG 检索：召回药物、miRNA 与调控机制相关文献或知识
6. 结果生成：输出排序结果、关联分数、机制证据和结果限制
7. 质量评测：通过 LLM-as-Judge 检查答案完整性与证据一致性
```

## 部署形态

```mermaid
flowchart LR
    CLIENT[客户端] --> DOCKER[Docker 部署]
    DOCKER --> API[Python / FastAPI 服务]
    API --> LLM[Claude / DeepSeek Compatible API]
    API --> REDIS[(Redis)]
    API --> CHROMA[(ChromaDB)]
    API --> MCP[MCP 工具服务]
    API --> SB[OpenSandbox]
```
