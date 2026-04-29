# AgentProof 调研报告：Agent 评价体系与可复用性分析

> 更新时间：2026-04-29  
> 研究范围：GitHub、Docs、Whitepaper、线上 API 行为、关键源码（oracle/indexer/contracts/sdk）

## 1. 调研目标

本报告聚焦三个问题：

1. AgentProof 如何“评价一个 Agent”（评分、风险、推荐、门控）  
2. 哪些能力可直接复用（API、SDK、代码、合约、方案）  
3. 对自建 Agent Reputation Index 平台的可落地借鉴

---

## 2. 一句话结论

AgentProof 的核心不是单一评分模型，而是一套 **Trust Oracle 基础设施**：

- 用多链索引与离线预计算生成信誉分
- 用 REST/MCP/A2A 对外提供可查询信任能力
- 用链上 Oracle + Gate/Hook 把“信任”变成可执行约束

即：**Score 是输入，Infrastructure 是产品。**

---

## 3. 评价体系（按实现链路拆解）

## 3.1 数据层（Indexing）

- 索引多链 ERC-8004 相关事件与业务数据
- 核心数据表/维度包括：agents、reputation_events、validation_records、score_history、failure_events、job_outcomes、deployer_reputation 等
- 架构上采用“周期任务 + 增量重算”

关键点：线上信任查询不是完全实时重算，而是依赖预计算结果。

## 3.2 评分层（Scoring）

### 3.2.1 实际线上主评分来源

从源码可见，Oracle 返回的 `composite_score` 主要采用 `agents` 表中预计算值（indexer 产物）作为 canonical score。

这意味着：

- 线上一致性好（前后端同源）
- 计算负担从请求时转移到后台任务
- 但会有“刷新周期”带来的时延

### 3.2.2 当前核心评分信号（indexer主路径，详细）

本节基于 `indexer/scoring.py` 与 `indexer/indexer.py` 的实现。  
评分流程可拆成四步：**取数 -> 预处理 -> 信号归一化 -> 加权合成**。

#### 第一步：原始数据从哪里来

1. **评分数据（rating）**  
   来源：`reputation_events.rating`  
   衍生字段：
   - `feedback_count = len(ratings)`
   - `average_rating = sum(ratings)/feedback_count`
   - `rating_std_dev = std(ratings)`

2. **verified 反馈计数**  
   来源：`reputation_events.verified`（布尔）  
   - `verified_feedback_count = count(verified is True)`
   - 未标记/False 视为 unverified

3. **验证数据（validation）**  
   来源：`validation_records.is_valid`（过滤 null）  
   - `completed = len(validation_records)`
   - `successful = count(is_valid=True)`
   - `validation_success_rate = successful/completed*100`（无数据时为 0，后续转中性分）

4. **账户年龄（age）**  
   来源：`agents.registered_at`  
   - 计算 `account_age_days = now - registered_at`

5. **在线率（uptime）**  
   来源：`uptime_daily_summary`  
   - 汇总最近窗口内成功/总检查，得到 `uptime_pct`（0~100）
   - 若无 uptime 数据则记为缺失

#### 3.2.2.A 字段字典（重构必备）

下面把“参与打分的字段”逐个展开，包含：**字段含义、来源、取值范围、缺失处理、在公式中的用途**。

| 变量名 | 含义 | 来源（表.列） | 取值/类型 | 缺失与边界处理 | 用途 |
|---|---|---|---|---|---|
| `agent_id` | Agent主键（ERC-8004 tokenId） | `agents.agent_id` | int | 必须存在；不存在则不评分 | 所有查询的过滤键 |
| `registered_at` | Agent注册时间 | `agents.registered_at` | ISO datetime | 解析失败会抛错；正常应为UTC时间戳 | 计算`account_age_days` |
| `canonical` | 用于年龄计算的“主记录” | `agents`同一`agent_id`的多行 | row | 取`registered_at`最早那条（跨链最早） | 防止跨链重复注册导致年龄偏小 |
| `ratings` | 所有评分值集合 | `reputation_events.rating`（按`agent_id`） | list[int] | 查询异常时退化为`[]` | 算均值、标准差、反馈量 |
| `feedback_count` | 原始反馈条数 | `len(ratings)` | int >= 0 | 空列表=0 | 参与tier判定、一致性分支 |
| `average_rating` | 原始平均评分 | `sum(ratings)/len(ratings)` | float 0~100 | 无评分时=0 | 进入贝叶斯平滑 |
| `rating_std_dev` | 评分标准差 | `calculate_std_dev(ratings)` | float >=0 | `<2`条时直接返回0 | 一致性信号输入 |
| `verified_feedback_count` | 已验证反馈数 | `reputation_events.verified` | int >=0 | 仅`verified is True`计数；缺列/异常时退化0 | 计算`effective_count` |
| `completed` | 有效验证记录数 | `validation_records.is_valid`（过滤null） | int >=0 | 查询异常时=0 | 验证成功率分母 |
| `successful` | 验证成功记录数 | 同上（`is_valid=True`） | int >=0 | 查询异常时=0 | 验证成功率分子 |
| `validation_success_rate` | 验证成功率 | `successful/completed*100` | float 0~100 | `completed=0`时先置0，后转中性50 | `validation_score` |
| `total_checks` | 可用性检查总次数（窗口） | `uptime_daily_summary.total_checks`（最近30条） | int >=0 | 没有记录则保持缺失态 | `uptime_pct`分母 |
| `successful_checks` | 成功检查次数（窗口） | `uptime_daily_summary.successful_checks` | int >=0 | 没有记录则保持缺失态 | `uptime_pct`分子 |
| `uptime_pct` | 在线率百分比 | `successful_checks/total_checks*100` | float 0~100 或 -1 | 无数据时=-1（哨兵值） | `uptime_score`输入 |
| `old_score` | 旧分数（用于趋势/快照） | `agents.composite_score` | float 0~100 | 缺失默认0 | 判断是否写`score_history` |
| `chains_present` | 当前agent出现在几条链 | 同`agent_id`在`agents`行数 | int >=1 | 至少1 | 写回`agents.chains_active` |

补充：实现里使用 `reputation_events.select("rating, verified")`。如果数据库还没升级到含`verified`字段的版本，可能导致该查询异常并触发退化分支。生产重构时建议先确保迁移完成。

#### 3.2.2.B 字段到变量的“最小重构流程”（伪代码）

```python
agent_rows = query agents where agent_id=?
canonical = row with min(registered_at)

rows = query reputation_events(rating, verified) where agent_id=?
ratings = [row.rating]
verified_count = count(row.verified is True)

feedback_count = len(ratings)
average_rating = mean(ratings) if ratings else 0
rating_std_dev = std(ratings) if len(ratings)>=2 else 0

validations = query validation_records(is_valid != null) where agent_id=?
completed = len(validations)
successful = count(is_valid is True)
validation_success_rate = successful/completed*100 if completed>0 else 0

age_days = now_utc - parse(canonical.registered_at)

uptime_rows = query uptime_daily_summary(total_checks, successful_checks) where agent_id=? order by summary_date desc limit 30
if uptime_rows and sum(total_checks)>0:
    uptime_pct = sum(successful_checks)/sum(total_checks)*100
else:
    uptime_pct = -1

composite = calculate_composite_score(
  average_rating, feedback_count, rating_std_dev,
  validation_success_rate, age_days, uptime_pct, verified_count
)
tier = determine_tier(composite, feedback_count)
```

#### 第二步：verified feedback 如何进入计分

先把反馈数量从“原始条数”变成“有效条数（effective_count）”：

- `verified` 权重 = 1.0
- `unverified` 权重 = 0.5

公式：

`effective_count = verified_count + 0.5 * unverified_count`

这一步的作用是：**不是把 unverified 丢掉，而是降低其影响力**，减少刷分对模型收敛速度的影响。

#### 第三步：6 个信号的计算方式

下列分数最终都归一到 0~100：

1. **平均评分信号（rating_score，贝叶斯平滑）**

- 先验：`prior = 50`
- 伪样本：`k = 3`

`rating_score = (average_rating * effective_count + prior * k) / (effective_count + k)`

解释：样本少时更接近 50，样本多时更接近真实均值。

2. **反馈量信号（volume_score，对数）**

`volume_score = min(100, log10(effective_count + 1) / log10(101) * 100)`

解释：反馈越多越可信，但边际收益递减（防止单纯堆量）。

3. **一致性信号（consistency_score）**

- 若 `feedback_count < 2`，直接给中性值 50
- 否则：

`consistency_score = max(0, 100 * (1 - rating_std_dev / 50))`

解释：波动越大，一致性越差。

4. **验证成功率信号（validation_score）**

- 若有验证记录：`validation_score = validation_success_rate`
- 若无验证记录：给中性值 50（不因缺失而惩罚）

5. **账户年龄信号（age_score，对数）**

`age_score = min(100, log10(account_age_days + 1) / log10(366) * 100)`

解释：账户越成熟分越高，但同样是对数增长，避免“年龄碾压”。

6. **在线率信号（uptime_score）**

- 若有数据：`uptime_score = uptime_pct`
- 若无数据：给中性值 50

#### 第四步：加权合成最终分数

当前实现权重：

- rating_score: 35%
- volume_score: 12%
- consistency_score: 13%
- validation_score: 18%
- age_score: 7%
- uptime_score: 15%

合成公式：

`composite = rating*0.35 + volume*0.12 + consistency*0.13 + validation*0.18 + age*0.07 + uptime*0.15`

最后处理：

- 截断到 `[0, 100]`
- 四舍五入到小数点后 2 位

---

### 3.2.3 什么是 verified feedback？ERC-8183 起什么作用？

#### 3.2.3.1 verified feedback 的定义

`verified feedback` 指：这条评分被证明锚定到一个**真实发生且已完成的链上工作**，而不是任意主观打分。

在 AgentProof 中，评分提交可携带 `job_id`；系统会去链上验证该 job 的完成事件是否存在。验证通过则该反馈记为 `verified=true`。

#### 3.2.3.2 ERC-8183 在这里的角色（核心）

AgentProof 使用 `AgentProofHook` 记录 `JobOutcomeRecorded(agentId, jobId, completed)` 事件。  
Oracle 在接收评分时会做校验：

1. 根据 `hook_chain + job_id + agent_id` 查询 Hook 事件日志
2. 必须存在 `completed=true` 的匹配事件
3. 通过后才能标记 `verified=true`（否则拒绝或记 unverified，取决于配置）

这就是 ERC-8183 的关键价值：把“评分”从**口碑声明**变成**服务完成证明**。

#### 3.2.3.2.A 验证链路中的字段字典（逐字段）

为方便重构，这里把“提交评分 -> 校验 -> 入库”的关键字段逐条列出：

| 字段 | 位置 | 含义 | 校验/规则 |
|---|---|---|---|
| `agent_id` | `POST /api/v1/feedback` body | 被评分的agent | 必填；数据库必须存在该agent |
| `rating` | body | 评分值 | 1~100 |
| `job_id` | body | ERC-8183作业ID | 可选；若开启强制模式则必填 |
| `chain` | body | 交互发生链 | 默认`base`；用于推断`hook_chain` |
| `hook_chain` | body | Hook事件所在链 | 默认回退到`chain` |
| `verified` | `reputation_events.verified` | 是否通过链上作业校验 | 仅当`job_id`可被链上证明且`completed=true`时为True |
| `hook_address` | `reputation_events.hook_address` | 使用的Hook合约地址 | 从配置`agentproof_hook_{chain}`解析 |
| `job_id` + `hook_chain` | `reputation_events` | 作业锚点联合键 | 同一作业仅允许一次评分（防重复） |
| `reviewer_address` | `reputation_events.reviewer_address` | 评分主体标识 | 由`api_key_id`哈希生成的伪地址 |
| `task_hash` | `reputation_events.task_hash` | 去重/关联任务 | 客户端不传则服务端自动生成 |
| `tx_hash` | `reputation_events.tx_hash` | API写入的伪交易哈希 | 服务端生成，用于审计追踪 |

链上验证使用的事件与topic过滤：

- 事件签名：`JobOutcomeRecorded(uint256 agentId, uint256 jobId, bool completed)`
- 过滤条件：`topic0=事件签名`、`topic1=agent_id`、`topic2=job_id`
- 必须命中最新日志且 `completed == true`

配置项（重构必须保留）：

- `agentproof_hook_{chain}`：每条链的Hook地址
- `hook_event_scan_blocks`：回扫区块窗口
- `feedback_require_job_id`：是否强制所有评分必须job锚定

#### 3.2.3.3 对最终得分有什么实际影响

verified 并不直接改变 `average_rating`，而是影响 `effective_count`，从而影响两件事：

1. 贝叶斯平滑收敛速度（`rating_score`）
2. 反馈量信号（`volume_score`）

直观例子（同样 10 条反馈）：

- 场景 A：10 条都 verified -> `effective_count = 10`
- 场景 B：10 条都 unverified -> `effective_count = 5`

在场景 B 下，模型会更“保守”，更靠近先验 50，volume 也更低，抗刷分能力更强。

#### 3.2.3.4 风控附加收益

除加权外，Job 锚定还带来：

- **反女巫**：刷评分需要先真实完成 job，成本上升
- **可追溯**：每条 verified 反馈可回查链上事件
- **可配置强约束**：`feedback_require_job_id=true` 时可强制只收 job 锚定评分

这使“信誉”从软信号升级为具备证据链的硬信号。

### 3.2.4 分层（Tier）

当前阈值：

- diamond: score >= 85 且 feedback >= 20
- platinum: >= 72 且 >= 10
- gold: >= 58 且 >= 5
- silver: >= 42 且 >= 3
- bronze: >= 30 且 >= 1
- unranked: 其余

## 3.3 风险层（Risk Flags）

风险标记与评分解耦，单独计算并返回。典型标记：

- HIGH_RISK_SCORE
- LOW_FEEDBACK / UNVERIFIED
- CONCENTRATED_FEEDBACK（单 reviewer 贡献 > 60%）
- LOW_UPTIME
- SERIAL_DEPLOYER
- FREQUENT_URI_CHANGES
- NEW_IDENTITY
- HIGH_FAILURE_RATE / ACTIVE_FAILURE
- HIGH_JOB_FAILURE_RATE / JOB_ABANDONMENT
- SUSPICIOUS_VOLATILITY

输出包含：

- `risk_flags`
- `risk_level`（low/medium/high/critical）
- `recommendation`（TRUSTED/CAUTION/HIGH_RISK/UNVERIFIED）

## 3.4 惩罚层（Penalty Registry）

支持“硬地板”强惩罚（override 组合分）：

- KNOWN_MALICIOUS / CONFIRMED_EXPLOIT / SANCTIONED_ADDRESS -> score floor = 0
- RUGPULL_ASSOCIATED -> score floor = 5

同时具备恢复机制（依据连续良好行为天数线性恢复）。

## 3.5 执行层（On-chain Trust Execution）

不是仅展示分数，而是可用于交易前门控：

- `TrustScoreOracle.sol`：多 Oracle 写入 + 共识平均 + 分歧检测
- `AgentProofHook.sol`：setProvider 前检查 score/tier/freshness/attestation
- `ReputationGateV2.sol`：协议内 `requireTrust` / value gating / collateral multiplier

---

## 4. 对外能力盘点（可集成资产）

## 4.1 REST API（可直接消费）

典型接口：

- `GET /api/v1/trust/{agent_id}`
- `GET /api/v1/trust/{agent_id}/risk`
- `POST /api/v1/trust/batch`
- `GET /api/v1/agents/trusted`
- `GET /api/v1/network/stats`

说明：

- 读接口有免费与计费分层
- 批量与高级能力需 API Key

## 4.2 MCP（可给 LLM Agent 直连）

`POST /mcp`，支持：

- `tools/list`
- `tools/call`

典型工具：

- evaluate_agent
- find_trusted_agents
- risk_check
- network_stats
- hook_gate_check
- resolve_address / resolve_ens

## 4.3 A2A（Agent-to-Agent）

- `GET /.well-known/agent.json`（Agent Card）
- `POST /a2a`（JSON-RPC）

适合在 Agent 协作网络里嵌入信誉查询。

## 4.4 Webhooks

支持分数变化、风险变化、tier变化等事件通知，便于协议侧自动化策略。

## 4.5 SDK

- TypeScript SDK（`@agentproof/sdk`）
- Python SDK（`agentproof`）

注意：SDK 许可条款与仓库主 README 的 MIT 口径存在差异（见第 6 节）。

---

## 5. 白皮书口径 vs 实现口径（必须关注）

观测到“叙事层”和“运行层”的差异：

- 白皮书强调 8~11 信号、更复杂的全量模型
- 实际线上主分数由 indexer 预计算公式提供（工程化版本）
- Oracle 层可能计算更细 breakdown，但 composite 以落库值为主

建议在对标时：

- **以源码与线上响应为准**
- 把白皮书作为路线图与产品叙事参考

---

## 6. 复用可行性与合规风险

## 6.1 可直接复用（低成本）

1. API 查询能力（trust/risk/batch/network）
2. MCP/A2A 协议接入方式
3. 风险旗标体系设计
4. “离线评分 + 在线查询 + 缓存 + 周期任务”工程范式

## 6.2 可改造复用（中成本）

1. 评分信号框架（权重重训）
2. reviewer 权重与 verified feedback 机制
3. deployer lineage 与 freshness penalty
4. 异常检测作业（波动、集中度、失效恢复）

## 6.3 需谨慎（高风险）

1. SDK 许可条款限制竞争性用途（需法务确认）
2. 直接复制其“品牌化评分标准”可能导致产品同质化
3. 白皮书指标与线上实现不一致可能引发预期管理问题

---

## 7. 对自建 Reputation Index 的落地建议

建议采用四层架构：

1. **Index Layer**：多源事件与行为数据归集  
2. **Scoring Layer**：可解释分数 + 分维度分数  
3. **Risk Layer**：独立风险旗标与级别  
4. **Execution Layer**：策略门控（链上/链下）

并优先实现以下 MVP 特性：

- `score + risk_flags + recommendation` 三元输出
- verified/unverified feedback 分权
- reviewer concentration 与 volatility 检测
- API 批量查询与 webhook 推送

---

## 8. 关键参考入口

- GitHub: https://github.com/BuilderBenv1/agentproof
- Docs: https://agentproof.sh/docs
- Whitepaper: https://agentproof.sh/whitepaper
- Website: https://agentproof.sh/
- Oracle API: https://oracle.agentproof.sh/api/v1
- Public API: https://api.agentproof.sh

