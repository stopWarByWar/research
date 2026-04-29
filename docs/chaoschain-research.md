# ChaosChain 调研报告：Agent 评估体系与可复用性分析

> 更新时间：2026-04-29  
> 研究范围：GitHub（ChaosChain/chaoschain）、官方 docs、官网 Worldline、核心工作流与合约实现

## 1. 调研目标

本报告聚焦三个问题：

1. ChaosChain/Worldline 如何评价一个 Agent（数据、评分、共识、发布）  
2. 哪些能力可直接复用（API、SDK、代码、方案）  
3. 对自建 Agent Reputation Index 平台有哪些可落地借鉴

---

## 2. 一句话结论

ChaosChain 的核心不是“单一评分函数”，而是一套 **Session 证据管线 + DKG 因果图 + 多维评分 + 信誉发布** 的工程系统。  
现阶段默认运行形态为 **off-chain-first（网关落库优先）**，链上主要承担结算/共识与可验证发布角色。

即：**可解释证据流是产品中枢，链上信誉是结算锚点。**

---

## 3. 宣传口径 vs 真实实现

## 3.1 对外口径（docs / web）

官网与 docs 核心叙事：

- Worldline 是 AI agent 的决策层，基于真实 coding session 评分
- 流程为 Capture → Analyze → Score → Compare
- 五维评分：reasoning、compliance、efficiency、collaboration、initiative
- 提供 MCP 与直接 API（start → step → submit → poll）

主要入口：

- `https://studio.chaoscha.in/docs`
- `https://studio.chaoscha.in/`

## 3.2 源码核验后的真实路径

基于仓库源码可确认：

1. 网关工作流负责把会话证据转为 DKG 与评分上下文  
2. 默认工作流为 off-chain-first，许多步骤“先落库完成”  
3. 合约层具备完整共识与 ERC-8004 发布能力，但与网关存在解耦

关键文件：

- `packages/gateway/src/workflows/work-submission.ts`
- `packages/gateway/src/workflows/score-submission.ts`
- `packages/gateway/src/services/dkg/engine.ts`
- `packages/contracts/src/libraries/Scoring.sol`
- `packages/contracts/src/RewardsDistributor.sol`

---

## 4. 评估体系（按实现链路拆解）

## 4.1 数据入口：Session API 与 Work 流程

Session API 提供标准化事件采集：

- `POST /v1/sessions`
- `POST /v1/sessions/:id/events`
- `POST /v1/sessions/:id/complete`
- `GET /v1/sessions/:id/context`
- `GET /v1/sessions/:id/evidence`

会话完成后，网关将事件物化为 Evidence DAG，再桥接 WorkSubmission 工作流。

## 4.2 DKG 计算：确定性纯函数

`packages/gateway/src/services/dkg/engine.ts` 显式强调纯函数约束：

- 同样 evidence 输入必须得到同样 DAG 与权重
- 无隐藏状态、无随机性、无外部依赖
- 通过稳定排序、可重复路径搜索保证确定性

产出：

- `thread_root`
- `evidence_root`
- 节点/边/roots/terminals
- contribution weights（betweenness 或 path_count）

## 4.3 信号提取：从图结构到特征

在 DKG 引擎内有可复用的特征提取逻辑（`extractPoAFeatures`）：

- initiative：root 作者占比
- collaboration：边参与度
- reasoning：可达深度归一化
- compliance / efficiency：可留空或依赖 policy 与 verifier 判断

这层是“可解释评分”的关键，因为每个维度都能回指图结构证据。

## 4.4 分数合成：信号 + 验证者判断

文档中以 `verifyWorkEvidence()` + `composeScoreVector()` 表达评分合成流程：

1. 先验证 evidence 并抽取 deterministic signals  
2. 再输出 5 维整数分（0~100）  
3. compliance、efficiency 常要求验证者显式给分

## 4.5 共识聚合：鲁棒统计（链上）

`packages/contracts/src/libraries/Scoring.sol` 使用：

1. 按维度算 stake-weighted median  
2. 计算 MAD（Median Absolute Deviation）  
3. 以 `alpha * MAD` 去异常值  
4. 对内点做 stake-weighted 平均

这是可直接复用的“抗异常验证者”共识模块。

## 4.6 信誉发布：ERC-8004 多维写入

`packages/contracts/src/RewardsDistributor.sol` 中：

- worker 每个维度单独 `giveFeedback`
- 维度标签使用字符串（如 Initiative/Collaboration/...）
- studio 地址作为第二标签（tag2）
- verifier 侧另有 `VALIDATOR_ACCURACY` 相关发布逻辑

---

## 5. “off-chain-first”现实影响（必须关注）

`work-submission.ts` 和 `score-submission.ts` 的默认路径体现为：

- WorkSubmission 可在网关侧持久化后即标记 completed
- ScoreSubmission 的 direct 模式也可直接落库并完成

影响：

1. 读 API 的“实时状态”主要来自网关数据库  
2. 链上不一定是每次评分的第一落点  
3. 产品体验更快，但“完全链上实时可验证”不是默认路径

这不是缺陷，而是架构取舍：先保证可用性、恢复性、吞吐，再做链上结算对齐。

---

## 6. 对外可集成能力盘点

## 6.1 公共读 API（可直接消费）

来自 `docs/PUBLIC_API_SPEC.md`：

- `GET /v1/agent/{agentId}/reputation`
- `GET /v1/work/{hash}`
- `GET /v1/work/{hash}/context`（通常需 API key）
- `GET /v1/studio/{address}/work`
- `GET /v1/studio/{address}/leaderboard`
- `GET /v1/work/{hash}/viewer`

## 6.2 Session 采集 API（强复用价值）

- 事件 schema 清晰，可作为你平台的数据入口标准
- 支持 context/evidence/viewer，一次满足“打分 + 可解释展示”

## 6.3 SDK 能力边界

`packages/sdk/README.md` 显示 Python SDK 已把多项能力迁至网关：

- DKG、XMTP、证据上传等能力逐步 gateway-first
- SDK更偏客户端编排与调用层

因此复用策略应是：**优先复用网关模式与接口，不要假设 SDK 本地就有全部核心逻辑**。

---

## 7. 复用性评估（面向自建 Reputation Index）

## 7.1 可直接复用（低成本）

1. Session 事件采集模型（start/events/complete）  
2. context/evidence 双视图 API 设计  
3. 5 维评分展示与场景化 compare（production/prototyping/review）  
4. leaderboard 聚合思路

## 7.2 可改造复用（中成本）

1. DKG 纯函数引擎（确定性与可追溯）  
2. 维度信号到分数的合成管线  
3. verifier 准确度信誉（meta-reputation）  
4. 多维标签化发布到信誉注册表

## 7.3 需谨慎（高成本/高风险）

1. 直接沿用其完整链上经济模型（需匹配你的激励与风险假设）  
2. 依赖其 hosted gateway（会引入供应商依赖）  
3. 文档口径与默认运行模式间的版本差异（需要持续核验）

---

## 8. 对你的平台的落地建议

建议采用四层架构：

1. **Evidence Layer**：Session 事件 + DAG 物化  
2. **Scoring Layer**：deterministic signals + verifier overrides  
3. **Consensus Layer**：MAD 鲁棒聚合  
4. **Publishing Layer**：多维信誉索引（链下主索引 + 链上锚定）

MVP 优先级：

- 先跑通 Session→DAG→5维分→leaderboard  
- 再接入 verifier accuracy  
- 最后接入链上锚定与结算

---

## 9. 关键参考入口

- GitHub: `https://github.com/ChaosChain/chaoschain`
- Docs: `https://studio.chaoscha.in/docs`
- Web: `https://studio.chaoscha.in/`
- API Spec: `docs/PUBLIC_API_SPEC.md`
- Verifier Guide: `docs/VERIFIER_INTEGRATION_GUIDE.md`

