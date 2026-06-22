# Sofia 文档整理：产品、概念与生态全景

> 整理时间：2026-06-22
> 来源：https://doc.sofia.intuition.box/
> 说明：本文是对 Sofia 官方文档站的结构化整理（中文）。原站点为单页应用，内容来自 `intro / manifesto / about / core-concepts / features / social / resonance / gamification / architecture / ecosystem / litepaper / known-issues` 等栏目。

---

## 0. 一句话概述

**Sofia（Semantic Organization for Intelligence Amplification）是一款 Chrome 浏览器扩展**：它把你日常的网页浏览，转化为可验证、链上认证、由你自己拥有的「信任知识图谱」。底层构建在 **Intuition 协议** 之上，隐私计算依托 **Phala TEE**，智能体框架使用 **Mastra**。

核心理念（来自 Manifesto）：**「网络在卖给你星星，我们卖给你动词。」**——信任不是一个数字（五星评分），而是一种用动词表达的关系（trusts / learning from / inspired by），并通过链上质押让表达「有代价、可携带、可验证」。

---

## 1. 文档站地图（栏目结构）

> 标注 `（未发布）` 的页面在 JS 路由中被隐藏（`ai-features/*` 与 `ecosystem/gaianet`），线上访问会 404，但内容已写好，属于草稿/路线图。

| 栏目 | 页面 | 主题 |
| --- | --- | --- |
| 入口 | Introduction (`/docs/intro`) | Sofia 是什么 |
| | Manifesto (`/docs/manifesto`) | 七条宣言：为什么不信星星 |
| | About us (`/docs/about`) | 团队与顾问 |
| Core Concepts | Atoms | Intuition 中最小数据单元 |
| | Triples & Vaults | 三元组与金库（质押/赎回） |
| | Predicates | 连接你与页面的「动词」 |
| Features | Getting Started | 安装、导入书签、追踪开关 |
| | Echoes | 按域名聚合的浏览历史 |
| | Certifications | 把访问行为提交上链 |
| | Intentions | 8 种意图类型 |
| | Bookmarks & Signals | 书签与信号 |
| AI Features（未发布） | Pulse Analysis | 分析打开的标签页并提取信号 |
| | Interest Analysis | 从认证中构建兴趣/专长画像 |
| | Chat & Predicates | 与 Sofia 对话、升级时生成谓词 |
| Social | Following & Trust Circle | 关注与信任圈（链上质押背书） |
| | Social Verification | 关联社交账号、Golden Profile |
| Resonance | Circle Feed | 信任圈实时认证流 |
| | Trending | 最热认证页面 |
| | Vote | 对主张（claims）投支持/反对 |
| | Featured Lists | 社区精选清单 |
| | Streak Leaderboard | 连续活跃排行榜 |
| Gamification | Currencies & Levels | XP / Gold 与用户、域名等级 |
| | Quests & Discovery | 61 个任务与发现奖励 |
| | Streaks & Voting | 连签与投票系统 |
| | Badges & Rewards | 徽章与 Beta 赛季奖池 |
| Architecture | Overview | 端到端系统架构 |
| Ecosystem | Intuition | 链上知识图谱（协议底座） |
| | Mastra | 智能体框架 |
| | Phala | TEE 隐私计算 |
| | GaiaNet（未发布） | 去中心化 AI 模型 |
| Litepaper | Introduction / Network / Subscription / DAO / Features / Privacy / Why Unique / Audience | 愿景、经济模型与治理 |
| Known Issues | Social Verification / Transactions | Alpha 阶段已知问题与规避 |

---

## 2. 核心概念（基于 Intuition）

Sofia 的数据模型直接复用 Intuition 协议的「原子—三元组」结构。

### 2.1 Atoms（原子）

原子是 Intuition 中的最小数据单元，是任意实体的唯一标识符（URL、用户、概念、想法）。

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| User | `0x1234...abcd` | 你的钱包地址 |
| URL | `https://github.com` | 你访问过的网页 |
| Predicate | `visits for work / trust / distrust` | 一种关系类型 |

首次认证一个页面时，Sofia 自动创建：URL 原子（若不存在）→ 谓词原子（意图类型）→ 用三元组把它们连接起来。**每个原子都有自己的金库（vault），用户可向其存入 TRUST 代币。**

### 2.2 Triples & Vaults（三元组与金库）

三元组结构为 **Subject → Predicate → Object**，例如：

| Subject | Predicate | Object |
| --- | --- | --- |
| alice.eth | visits for learning | docs.python.org |
| bob.eth | visits for work | github.com/project |

每个三元组具有：Vault ID（链上唯一标识）、Creator、Deposits（累计质押的 TRUST）、Shares（金库份额）。

**质押 / 赎回机制（bonding curve）**：认证页面时把 TRUST 存入三元组金库 → 获得与存入额成比例的份额 → 早期存入者获得更多份额（早鸟优势）→ 之后可赎回份额换回 TRUST，若后来者继续存入则可能获利。页面被越多人认证，其金库越有价值；做「Pioneer（先驱）」可用同样的 TRUST 获得更多份额。

完整认证流程：
```
1. 访问 github.com/awesome-project
2. 点击「Learning」进行认证
3. Sofia 创建/查找原子：A=你的钱包，B="visits for learning"，C=URL
4. 创建三元组 A → B → C
5. 向金库存入 0.1 TRUST
6. 获得金库份额
7. 该认证永久上链
```

### 2.3 Predicates（谓词）

谓词即连接「你」与「页面」的动词，每种意图对应一个谓词：

| 意图 | 谓词 |
| --- | --- |
| Work | "visits for work" |
| Learning | "visits for learning" |
| Fun | "visits for fun" |
| Inspiration | "visits for inspiration" |
| Buying | "visits for buying" |
| Music | "visits for music" |
| Trusted | "trusts" |
| Distrusted | "distrusts" |

> 官方注明：核心团队正在扩展更多谓词类型。

---

## 3. 核心功能（User Guide）

### 3.1 Getting Started

- **导入 Chrome 书签**：首次启动可导入现有书签，填充 Echoes。
- **追踪开关**：可在设置里随时启停；开启时仅记录你「主动浏览满 3 秒以上」的页面。
- **隐私过滤**：登录页、广告、追踪像素会被自动过滤。

### 3.2 Echoes（回声）

把浏览历史按域名智能聚合，而非线性 URL 列表。

- 属性：Domain、Pages、Level（1–10）、Certifications、Last Visit。
- 页面状态：`Visited`（白色，未认证）/ `Certified`（带意图颜色圆点 + 绿色边框）。
- 域名归一化：`www.github.com → github.com`、`m.facebook.com → facebook.com`、`app.slack.com → slack.com`。
- 动作：排序、筛选、打开、Level Up（花 Gold 升级，AI 生成新谓词）、Amplify（上链发布「我 [谓词] [域名]」，成本 0.01 TRUST + 手续费）、删除、移除 URL（已认证则回收质押）。

### 3.3 Certifications（认证）

认证 = 把一次页面访问连同「意图 + TRUST 质押」提交上链。

| 参数 | 说明 | 默认 |
| --- | --- | --- |
| Amount | 存入的 TRUST | 0.01 TRUST |
| Pool Split | 进入 Beta 赛季奖池的比例 | 20% |
| Intention | 访问理由 | 必填 |

链上发生：创建 URL 原子（若新）→ 创建三元组 → 锁定 TRUST 到金库 → 发放份额。
重复认证：相同意图 = 增加质押；不同意图 = 新建三元组。

**费用结构（Weight Modal 中展示）**：

| 费用 | 说明 |
| --- | --- |
| Protocol Fee | Intuition 网络费（0.2 TRUST） |
| Sofia Fee | 平台费（5% + 0.01 TRUST） |
| Gas | 链上交易费 |

### 3.4 Intentions（8 种意图）

每次认证必须选一个意图，为认证赋予语义：

| 意图 | 用途 | 示例 |
| --- | --- | --- |
| Work | 专业任务 | 文档、工具、仓库 |
| Learning | 教育成长 | 教程、课程 |
| Fun | 娱乐 | 视频、游戏、社媒 |
| Inspiration | 创意灵感 | 设计、作品集 |
| Buying | 购物 | 电商、测评、优惠 |
| Music | 音乐 | Spotify、SoundCloud |
| Trusted | 信任来源 | 可靠参考 |
| Distrusted | 不信任来源 | 不实/不可靠内容 |

语义层带来：兴趣映射（区分学习 vs 工作）、信任信号（标记信/不信）、画像构建（专长自动浮现）。

### 3.5 Bookmarks & Signals

书签可按意图列表组织，并在个人资料中对其他用户可见。

---

## 4. 社交（Social）

### 4.1 Following & Trust Circle（关注与信任圈）

- **Following**：关注他人，跟踪其认证、建立网络。
- **Trust Circle（信任圈）**：通过质押 TRUST 在链上为他人背书。加入流程：在三元组金库（如 `I - Trust - Wieedze.eth`）存入 TRUST → 获得份额 → 公开表达支持 → 之后可赎回（可能获利）→ 对方的认证会出现在你的 Resonance Circle Feed 中。
- 链上声誉随每次认证、每个关注者、每次成功策展而增长，**质量与一致性比数量更重要**。

### 4.2 Social Verification（社交验证）

关联社交账号以验证身份并赚取奖励：

| 平台 | 方式 | 奖励 |
| --- | --- | --- |
| Twitter/X、Discord、YouTube、Twitch、Spotify | OAuth 连接 | 各 100 XP |

**Golden Profile**：验证全部 5 个平台后解锁——头像绿色边框、+100 XP、网络中更高可信度。
> Alpha 限制：Spotify 与 Twitter/X 验证暂未对所有用户开放（详见已知问题）。

---

## 5. 共鸣（Resonance）社区层

| 页面 | 作用 |
| --- | --- |
| **Circle Feed** | 信任圈的实时认证流；可点赞/点踩（每票 1 TRUST）、按意图筛选、访问他人资料 |
| **Trending** | 当前最多认证的页面；按意图筛选、快速打开、直接认证 |
| **Vote** | 对 Sofia Claims / Featured Claims 投支持或反对（自定义 TRUST，可选 Linear / Progressive 曲线）。示例主张：「Claude 比 ChatGPT 好」「披萨该放菠萝」 |
| **Featured Lists** | 来自 Intuition 社区的精选清单（如「Top Agent Skills」「Best AI Code Editors」），可对条目质押支持/反对并赚取奖励 |
| **Streak Leaderboard** | 按「认证连签 / 投票连签」排名，可对比位置、访问榜首、追踪进度 |

---

## 6. 游戏化（Gamification）

### 6.1 货币与等级

两种货币：

| | XP（经验值） | Gold（金币） |
| --- | --- | --- |
| 来源 | 完成任务 | 认证(+10)、发现奖励、投票(+5/票) |
| 用途 | 决定用户等级 | 升级域名等级 |
| 存储 | 链上（可验证） | 链下（本地） |
| 可交易 | 否 | 否 |

Gold 公式：`totalGold = certificationGold + discoveryGold + voteGold - spentGold`

**域名等级（1–10）**，认证阈值：`[0, 3, 7, 12, 18, 25, 33, 42, 52, 63, 75]`，升级需达到阈值并花费 Gold（如 1→2 需 3 次认证 + 30 Gold），并获得 XP 奖励。

### 6.2 任务与发现

- **61 个任务**奖励 XP，需上链领取。示例：First Signal(50 XP)、Daily Certification(10 XP)、Domain Explorer(30 XP)、Intention Master(40 XP)、Heavy Hitter(50 XP)、Committed Streak(75 XP)、Social Linked Quest(100 XP) 等；每日任务午夜重置。
- **发现奖励（Discovery Gold）**：率先认证某页面可得——1st = Pioneer(+50)、2nd–10th = Explorer(+20)、11th+ = Contributor(+10)。

### 6.3 连签与投票

- **Streaks**：每天至少 1 次认证则连签 +1，断签清零；连签数显示在排行榜。即便 0.01 TRUST 的小认证也算数，周末别断。
- **Voting**：对认证投票 1 TRUST/+5 Gold；对主张投票自定义 TRUST/+5 Gold。每日最多 10 票、投票 Gold 上限 50/天。

### 6.4 徽章与奖池

- **Badges**：Milestones、Streaks、Social、Pioneer、Expertise 等类别。
- **Beta Season Pool**：每笔存款 20% 进入共享奖池（80% 进自己的金库），赛季末可按贡献赎回份额。
- 进阶建议：每日做任务、抢做 Pioneer、关联社交账号、聚焦喜欢的域名升级、每日投票稳定赚 Gold。

---

## 7. 系统架构（Architecture）

围绕**隐私、去中心化、可验证知识**设计，分层如下：

1. **Chrome 扩展（用户层）**：监测浏览、捕获 URL 与上下文，数据存本地 localStorage，**仅发送至 Phala TEE**。
2. **Phala TEE（隐私层）**：所有敏感数据处理在可信执行环境中完成，连基础设施提供方都无法读取原始数据；端到端加密 + 链上证明公开验证。
3. **Mastra 框架（运行于 TEE 内）**：AI 智能体分析浏览模式、提取信号、构建知识图谱。
4. **GaiaNet（AI 模型）**：去中心化 AI 模型，处理来自 Phala 的匿名数据，保证可扩展性。
5. **Intuition MCP & Indexer（知识层）**：MCP 在 TEE 内执行，Sofia 智能体通过 MCP 查询/过滤用户知识图谱。

**数据流**：Capture（捕获）→ Process（TEE 安全处理）→ Analyze（智能体在 TEE 内分析）→ Anonymize（离开 TEE 前匿名化）→ Index（经 Intuition 上链索引）→ Query（通过智能合约交互）。

---

## 8. 生态系统（Ecosystem）

### 8.1 Intuition —— 链上知识图谱（底座）

- **知识三元组**：以 subject-predicate-object 存储信息，构成语义图谱。
- **Attestations**：他人可对你的信号背书，增强可信度。
- **Identity Atoms**：每个实体都有唯一链上标识。
- **Staking**：在信号上质押代币，使经济利益与真实性对齐。
- **Composability**：知识图谱可与其他应用组合，催生信任型 dApp。
- Sofia 智能体在 TEE 内处理活动后，生成匿名信号提交到 Intuition，形成「基于证明而非声明」的去中心化声誉系统（兴趣、技能、网络、贡献）。通过 UserWallet（ERC-4337）无缝处理创建原子/三元组、订阅与费用、背书、查询、访问控制等。

### 8.2 Mastra —— 智能体框架

- 能力：上下文理解、模式识别、信号生成（intuition 三元组）、推荐引擎、隐私优先（处理全在 TEE 内）。
- 多智能体架构：Content Analysis / Social Connection / Skill Validation / Interest Mapping / Recommendation 等专职智能体协同。
- 选择 Mastra 的原因：TypeScript 原生，与浏览器扩展共享代码库，便于快速迭代、生产级安全与可观测性。

### 8.3 Phala —— 可信执行环境（TEE）

- 价值主张：Zero Trust 架构、数据主权、合规就绪（GDPR/CCPA by design）、兼容去中心化、可验证计算（可在不访问数据的前提下验证计算正确性）。
- Phala 基础设施：基于 Intel SGX / ARM TrustZone 的硬件级安全、多节点去中心化计算、密码学证明、节点经济激励。可通过 Sofia Trust Center 查看可验证的证明报告。

### 8.4 GaiaNet（未发布）

去中心化 AI 模型，为 Sofia 智能能力提供算力与可扩展性（当前线上隐藏）。

---

## 9. Litepaper（愿景与经济模型）

### 9.1 问题与方案

- **问题**：当今互联网建立在信息不对称之上——不透明推荐、自我声明且易伪造的身份、被囚禁的数据、用户付出却零回报。
- **方案**：把浏览行为变成你拥有的、可认证、可携带、可变现的资产。每次认证生成上链语义三元组，累积成个人知识图谱，并经社区验证（投票、质押、辩论）成为「可验证凭证」。

### 9.2 核心机制

| 机制 | 角色 |
| --- | --- |
| Semantic Triples | 存储在 Intuition 的结构化关系 |
| Token Economy | TRUST 质押 + bonding curve 早鸟激励 |
| AI Agents | GaiaNet + Mastra 驱动的个人 AI |
| DAO Governance | 通过 Colony 进行声誉加权治理 |

### 9.3 基于「证明」的网络

技能/偏好/关系不只是声明，而是由你和社区验证。Sofia 还会**自动创建缺失信号**连接数据（如「我关注 Kendrick Lamar」→「Kendrick Lamar 是艺术家」→「艺术家在 YouTube 上」），丰富全局图谱、改善推荐。

### 9.4 订阅模型

订阅即激活与 MetaMask 关联的 **UserWallet（ERC-4337）**，用于自动覆盖运营成本（链上信号创建/注册、GaiaNet API 调用）。用户完全拥有资金，可随时充值、调整、取消。

### 9.5 DAO（Colony 治理）

- **按域与角色治理**：分为开发、营销、金库、合作等半自治域。
- **声誉为核心**：非「1 token = 1 票」，声誉靠贡献获得、不可购买、闲置衰减，防止巨鲸集权。
- **Lazy Voting（异议共识）**：仅在有可信异议时才投票；无异议自动通过，有异议则进入域内声誉加权投票；金额/风险越高，异议窗口或声誉门槛越高。
- **去中心化金库**：所有费用（订阅费、服务费）进入由社区管理的 DAO 金库，治理协议升级、合作、拨款等。

### 9.6 隐私与数据控制（Privacy Policy，更新于 2026-06-03）

- **本地优先**：开启追踪时仅在本地（IndexedDB / 扩展存储）记录 URL、标题、favicon、时间戳；大型黑名单过滤敏感网址、剥离敏感查询参数。
- **钱包与链上身份**：连接钱包后用公开地址读写 Intuition；任何认证都是你自己触发的公开、永久交易。
- **账户信息**：认证由 Privy 处理；可选连接外部平台的 access token 由后端安全存储，仅用于计算声誉信号。
- **共享对象**：Intuition（公链 + indexer）、IPFS（原子元数据 pin 到公共网络）、Privy（认证）；链上/IPFS 数据按设计公开且永久。
- **不出售数据、不用于广告**；可随时关闭追踪、清除本地数据、卸载扩展；上链需明确授权。

### 9.7 为何独特 & 目标人群

- **独特性**：基于行为而非声明、可验证凭证、语义知识图谱、隐私优先、贡献优先的精英治理。
- **目标人群**：想夺回数据控制权、想要真正个性化推荐、希望无需自我营销地展示技能、想发现真正契合的人与内容的用户——创作者、开发者、研究者，以及重视隐私与真实连接的人。

---

## 10. 已知问题（Alpha）

| 类别 | 问题 | 处理 |
| --- | --- | --- |
| YouTube 验证 | 可能失败 | 邮箱需核心团队后台手动配置 |
| Spotify 验证 | 不稳定/不可用 | Spotify OAuth 限制，团队处理中；可先验证另外 4 个平台拿 Golden Profile |
| Twitter/X 验证 | 非所有用户可用 | API 限额，暂无规避，待规模扩大后解决 |
| 交易失败（刷新后） | 刷新扩展后用旧标签页发交易会失败 | 关闭旧标签、用新标签页认证 |
| 交易失败（网络错误） | 无明确报错 | 确认钱包已切到 Intuition Mainnet |

> 反馈请在专用 Discord 频道，切勿私信核心团队。

---

## 11. 团队（About）

- **Samuel Chauche — 联合创始人 / CEO**：6 年企业技术经验（Credit Suisse、Airbus、Salesforce），Intuition 协议核心贡献者，Intuition 黑客松前 3。
- **Maxime Saint-Joannis — 联合创始人 / CTO**：14 年音乐制作人，搭建 Sofia 全部技术基础设施，全栈工程师，Intuition 黑客松前 5。
- **顾问**：Jeremie Olivier（Zet.box，导师）、James Woods（营销顾问，W O O D S）、Billy Luentke（产品布道者，0xBilly）。

---

## 12. 关键要点小结

1. **定位**：Sofia 是把「浏览行为 → 链上可验证信任图谱」的 Chrome 扩展，主打「用动词表达信任、用质押让信任有代价」。
2. **三层数据模型**：Atoms（实体）→ Predicates（动词）→ Triples（三元组 + 金库质押），完全复用 Intuition 协议。
3. **隐私架构**：本地优先 + Phala TEE + 离开前匿名化 + 仅经用户授权上链，是其差异化卖点。
4. **激励飞轮**：意图认证 → XP/Gold → 任务/连签/发现奖励 → 徽章/奖池，配合 bonding curve 早鸟优势驱动持续参与。
5. **治理与经济**：ERC-4337 订阅钱包覆盖运营成本；Colony 声誉加权 + Lazy Voting 实现精英治理。
6. **成熟度**：处于 Alpha；`ai-features/*` 与 `ecosystem/gaianet` 等页面已写好但线上未发布，部分社交验证受第三方 API 限制。

---

## 13. 参考入口

- 文档首页：https://doc.sofia.intuition.box/
- Introduction：https://doc.sofia.intuition.box/docs/intro
- Manifesto：https://doc.sofia.intuition.box/docs/manifesto
- Litepaper：https://doc.sofia.intuition.box/docs/litepaper/introduction
- Architecture：https://doc.sofia.intuition.box/docs/architecture/overview
