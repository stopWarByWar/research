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

### 3.2.2 当前核心评分信号（indexer主路径）

主路径为 6 信号加权：

1. 平均评分（贝叶斯平滑）
2. 反馈量（log）
3. 评分一致性（标准差反向）
4. 验证成功率
5. 账户年龄（log）
6. 在线率（uptime）

并引入 **verified feedback 权重**（ERC-8183 Job 锚定）：

- verified = 1.0
- unverified = 0.5

这对抗“无凭证刷分”非常关键。

### 3.2.3 分层（Tier）

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

