# ChaosChain 调研后的下一步落地蓝图（字段映射 + 数据模型）

> 更新时间：2026-04-29  
> 目标：将 ChaosChain 的方法论落到你自己的 Agent Reputation Index 平台

## 1. 下一步目标（工程可执行）

基于 `docs/chaoschain-research.md`，下一步建议直接进入“可开发状态”的三件事：

1. 定义标准化数据模型（session/evidence/score/reputation）  
2. 建立字段映射（ChaosChain API -> 内部统一模型）  
3. 固化最小闭环 pipeline（采集 -> 评分 -> 聚合 -> 查询）

---

## 2. 字段映射（ChaosChain -> 内部模型）

## 2.1 Session 元数据映射

| ChaosChain 字段 | 来源 | 内部字段建议 | 说明 |
|---|---|---|---|
| `session_id` | `/v1/sessions` | `sessions.session_id` | 业务主键 |
| `studio_address` | `/v1/sessions` | `sessions.namespace_id` | 建议抽象为 namespace/workspace |
| `agent_address` | `/v1/sessions` | `sessions.agent_instance_id` | 可映射到 agent_instance |
| `task_type` | `/v1/sessions` | `sessions.task_type` | feature/bugfix/refactor/general |
| `work_mandate_id` | `/v1/sessions` | `sessions.task_policy_id` | 策略版本锚点 |
| `status` | `/v1/sessions/:id/complete` | `sessions.status` | running/completed/failed |
| `workflow_id` | `/v1/sessions/:id/complete` | `sessions.pipeline_run_id` | 下游 pipeline 追踪 |
| `data_hash` | `/v1/sessions/:id/complete` | `sessions.evidence_commitment` | 证据承诺哈希 |

## 2.2 Session Event 映射

| ChaosChain 字段 | 来源 | 内部字段建议 | 说明 |
|---|---|---|---|
| `event_id` | `POST /v1/sessions/:id/events` | `session_events.event_id` | 幂等键 |
| `event_type` | 同上 | `session_events.event_type` | 统一枚举 |
| `timestamp` | 同上 | `session_events.event_ts` | ISO 时间 |
| `summary` | 同上 | `session_events.summary` | 人类可读摘要 |
| `causality.parent_event_ids[]` | 同上 | `session_event_edges.parent_event_id` | 单独边表 |
| `agent.role` | 同上 | `session_events.actor_role` | worker/verifier/collaborator |
| `agent.agent_address` | 同上 | `session_events.actor_id` | actor 标识 |

## 2.3 Evidence DAG 映射

| ChaosChain 字段 | 来源 | 内部字段建议 | 说明 |
|---|---|---|---|
| `nodes[].node_id` 或 `id` | `/v1/sessions/:id/evidence` | `evidence_nodes.node_id` | 节点主键 |
| `nodes[].parent_ids[]` | 同上 | `evidence_edges` | DAG 边 |
| `nodes[].agent_address`/`author` | 同上 | `evidence_nodes.author_id` | 贡献者 |
| `nodes[].payload_hash` | 同上 | `evidence_nodes.payload_hash` | 内容哈希 |
| `artifacts[]`/`artifact_ids[]` | 同上 | `evidence_artifacts` | 文件/对象关联 |
| `merkle_root` | 同上 | `evidence_graphs.merkle_root` | 图承诺 |
| `roots[]`,`terminals[]` | 同上 | `evidence_graphs.roots_json`,`terminals_json` | 可先 JSON 存储 |

## 2.4 分数与信誉映射

| ChaosChain 字段 | 来源 | 内部字段建议 | 说明 |
|---|---|---|---|
| `scores[0..4]` | score-submission | `score_vectors` | 对应 5 维 |
| `avg_scores` | leaderboard | `score_aggregates` | 会话/代理聚合 |
| `trust_score` | `/v1/agent/{id}/reputation` | `reputation_snapshots.trust_score` | 0~100 |
| `quality_score` | 同上 | `reputation_snapshots.quality_score` | 0~1 |
| `consensus_accuracy` | 同上 | `reputation_snapshots.validator_accuracy` | 0~1 |
| `evidence_anchor` | 同上 | `reputation_snapshots.evidence_anchor` | 证据定位 |
| `derivation_root` | 同上 | `reputation_snapshots.derivation_root` | 推导根 |

---

## 3. 建议数据库模型（MVP）

以下为推荐最小表集合（可直接建表）：

1. `sessions`  
2. `session_events`  
3. `session_event_edges`  
4. `evidence_graphs`  
5. `evidence_nodes`  
6. `evidence_edges`  
7. `work_submissions`  
8. `score_submissions`  
9. `score_consensus`  
10. `reputation_snapshots`

## 3.1 建表示例（PostgreSQL）

```sql
create table sessions (
  session_id text primary key,
  namespace_id text not null,
  agent_instance_id text not null,
  task_type text not null,
  task_policy_id text not null,
  status text not null check (status in ('running','completed','failed')),
  pipeline_run_id text,
  evidence_commitment text,
  started_at timestamptz not null,
  completed_at timestamptz
);

create table session_events (
  event_id text primary key,
  session_id text not null references sessions(session_id) on delete cascade,
  event_type text not null,
  event_ts timestamptz not null,
  actor_id text not null,
  actor_role text not null,
  summary text not null,
  payload jsonb not null default '{}'::jsonb
);

create table session_event_edges (
  session_id text not null,
  parent_event_id text not null,
  child_event_id text not null,
  primary key (session_id, parent_event_id, child_event_id),
  foreign key (parent_event_id) references session_events(event_id) on delete cascade,
  foreign key (child_event_id) references session_events(event_id) on delete cascade
);

create table evidence_graphs (
  session_id text primary key references sessions(session_id) on delete cascade,
  merkle_root text not null,
  roots_json jsonb not null,
  terminals_json jsonb not null,
  node_count int not null,
  edge_count int not null,
  created_at timestamptz not null default now()
);

create table evidence_nodes (
  node_id text primary key,
  session_id text not null references sessions(session_id) on delete cascade,
  author_id text not null,
  node_ts timestamptz not null,
  payload_hash text not null,
  canonical_hash text,
  metadata jsonb not null default '{}'::jsonb
);

create table evidence_edges (
  session_id text not null,
  from_node_id text not null references evidence_nodes(node_id) on delete cascade,
  to_node_id text not null references evidence_nodes(node_id) on delete cascade,
  primary key (session_id, from_node_id, to_node_id)
);

create table work_submissions (
  work_id text primary key,
  session_id text references sessions(session_id),
  namespace_id text not null,
  worker_id text not null,
  epoch bigint,
  status text not null check (status in ('pending','scored','finalized')),
  evidence_anchor text,
  derivation_root text,
  submitted_at timestamptz not null default now()
);

create table score_submissions (
  id bigserial primary key,
  work_id text not null references work_submissions(work_id) on delete cascade,
  worker_id text not null,
  verifier_id text not null,
  initiative smallint not null check (initiative between 0 and 100),
  collaboration smallint not null check (collaboration between 0 and 100),
  reasoning smallint not null check (reasoning between 0 and 100),
  compliance smallint not null check (compliance between 0 and 100),
  efficiency smallint not null check (efficiency between 0 and 100),
  raw_signals jsonb,
  submitted_at timestamptz not null default now(),
  unique (work_id, worker_id, verifier_id)
);

create table score_consensus (
  work_id text not null references work_submissions(work_id) on delete cascade,
  worker_id text not null,
  initiative smallint not null,
  collaboration smallint not null,
  reasoning smallint not null,
  compliance smallint not null,
  efficiency smallint not null,
  method text not null default 'median_mad_stake_weighted',
  participant_count int not null,
  finalized_at timestamptz not null default now(),
  primary key (work_id, worker_id)
);

create table reputation_snapshots (
  id bigserial primary key,
  agent_id text not null,
  trust_score numeric(5,2) not null,
  quality_score numeric(6,4),
  validator_accuracy numeric(6,4),
  evidence_anchor text,
  derivation_root text,
  source text not null default 'consensus_pipeline',
  snapshot_at timestamptz not null default now()
);
```

---

## 4. 核心索引与性能建议

```sql
create index idx_sessions_agent_status on sessions(agent_instance_id, status);
create index idx_events_session_ts on session_events(session_id, event_ts);
create index idx_nodes_session_author on evidence_nodes(session_id, author_id);
create index idx_work_namespace_status on work_submissions(namespace_id, status);
create index idx_scores_work_worker on score_submissions(work_id, worker_id);
create index idx_reputation_agent_time on reputation_snapshots(agent_id, snapshot_at desc);
```

---

## 5. Pipeline 设计（最小可运行）

## 5.1 Ingestion

1. 接收 session start/events/complete  
2. 落 `sessions` / `session_events` / `session_event_edges`  
3. 异步物化 `evidence_graphs` / `evidence_nodes` / `evidence_edges`

## 5.2 Scoring

1. 从 evidence graph 提取 deterministic signals  
2. 合成五维分（允许 verifier override）  
3. 写入 `score_submissions`

## 5.3 Consensus + Reputation

1. 按 `work_id + worker_id` 运行 MAD 聚合  
2. 写入 `score_consensus`  
3. 生成 `reputation_snapshots`（可选链上锚定）

---

## 6. API 最小集合（供你先实现）

1. `POST /v1/sessions`  
2. `POST /v1/sessions/:id/events`  
3. `POST /v1/sessions/:id/complete`  
4. `GET /v1/sessions/:id/evidence`  
5. `POST /v1/scores`（verifier 提交）  
6. `POST /v1/consensus/run`（内部或管理员）  
7. `GET /v1/agents/:id/reputation`  
8. `GET /v1/studios/:id/leaderboard`

---

## 7. 验收标准（技术口径）

完成下一步时，应满足：

1. 单 session 可从事件重建 DAG，且 `merkle_root` 可复算  
2. 同一输入证据重复计算得到一致 signals  
3. 多 verifier 分数可稳定聚合到共识结果  
4. reputation 接口可返回 trust_score + evidence_anchor + derivation_root

---

## 8. 实施顺序建议

1. 先建表 + 写入链路（不做复杂评分）  
2. 再接 deterministic signals（initiative/collaboration/reasoning）  
3. 再接 verifier 评分与 MAD 共识  
4. 最后接 reputation 查询与可视化

这样可以最快形成一个“可解释、可查询、可扩展”的最小产品闭环。

