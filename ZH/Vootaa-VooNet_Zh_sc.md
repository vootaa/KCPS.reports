# Vootaa — VooNet 灵感白皮书（精要版）

**版本**：初稿  
**目标读者**：产品 / 工程 / 社区决策者、潜在合作者与早期用户  
**风格**：技术与产品并重，面向快速落地的艺术体验链

## 摘要
Vootaa 是基于 Chainweb + Pact + 原生 SPV 的艺术体验区块链项目。网络名 VooNet，节点名 Voonet-Node。首发采用三链架构（Past / Now / Future），以故事化 3D 体验为核心：过去（纪念）、当下（交易/铸造/流通）、未来（孵化/试验）。用户对外只感知 Now 链（简化 UX），Archive（Past）与 Lab（Future）通过单向 SPV 证明与 Now 交互。目标以最小工程成本实现被采用的艺术 + 社区平台，并保留向多链扩展能力。

---

## 愿景与定位
- 愿景：用“可验证的链上仪式”将区块链记忆、艺术与实验结合，提供安全、简洁、有情感的链上体验。  
- 定位：艺术体验链 + Proof-as-a-Service。首发以艺术/纪念/社区吸引用户，随后开放孵化功能模块支持企业/开发者用例（IoT / AI / DePIN）。

---

## 三链角色与设计原则
**总体原则**：玩法差异化 + 功能对称备份（运维可切换）

### Archive — Past（废墟 / 记忆）
- 主题：纪念、回忆、KDA 社区历史资产迁移与展览。  
- 功能：迁移 Kadena Pact 合约、纪念 NFT、展览/拍卖，curator-only 写权限。  
- 对外策略：只读 / 受限写入。用户通过 Now 提交 proof 兑换/互动。

### Now — Present（当下 / 勇气）
- 主题：主要对外链，用户钱包与日常交互入口。  
- 功能：主用 Coin（基础铸币）、简化 DEX、Fuse/Claim 合约、用户交易与最终结算点。  
- 对外策略：唯一面向普通用户的链，UI/SDK 仅与 Now 交互。

### Lab — Future（未来 / 希望）
- 主题：试验、孵化、Proof-as-a-Service PoC（IoT anchors、AI proofs 等）。  
- 功能：沙箱合约、孵化项目部署区。成熟项目单向提交 proof 到 Now 触发兑换/激励。  
- 对外策略：开发者/合作伙伴可访问，普通用户通过 Now 参与。

---

## 跨链与 SPV 策略
- 单向可信提交：Archive → Now，Lab → Now，仅在 Now 合约中使用 Pact 的 verify-spv 验证 proofs。  
- 验证流程：Proof Service 在 Archive/Lab 上预计算 inclusion proof → 生成 proofId + blob → 用户或服务将 proof 提交至 Now 的 Fuse/Claim 合约 → 合约内 verify-spv 并做 replay 防护（proofId 一次性标记）。  
- 设计权衡：将复杂度与证明成本推给 Proof Service 与后台，合约保持最小化可验证逻辑。

---

## 经济与代币设计
- 三基础 Coin：CoinA / CoinB / CoinC（各链本地挖掘/分发，象征守护神），可流通与燃烧。  
- Essence（统一 Token / 稀有 NFT）：通过 Fuse（例如 1A+1B+1C 或可变配方）在 Now 铸造，作为消费/展览/稀有证明。  
- 发行模型（示例建议，最终以治理决定）：
  - 初始空投：守护神选择者与 KDA 社区老用户空投少量基础 Coin（Archive/Now 分配）。  
  - Fuse 条件：默认 1/1/1 简化上手，活动期间可调整配方。  
  - 费用与回收：Fuse/Claim 可能收取小额手续费，用于网络运维与 curator 基金。  
  - DEX/流动性：Now 上提供简化交换，高阶市场建议链下/托管市场。

---

## 核心组件与技术实现
- 节点：Voonet-Node（基于 Chainweb node 2.31.1 精简配置，Graph 可用 3-node 全连或环）。  
- Pact：首发使用 Pact5（并发、verify-spv 支持），合约包括基础 Coin、Fuse/Claim、NFT、Audits 表。  
- SPV Proof Service：在 Archive/Lab 上运行 worker 预计算 proofs，提供 REST API、proofId、压缩/签名与付费队列。  
- VCL（Virtual Chain Layer）：单一对外 API 层（submit / getProof / verify），隐藏多链细节并对 Now 做最终提交（可选代理 proof 提交）。  
- Edge Gateway：前端事件聚合、batch merkleRoot 生成、rate-limit、签名校验。  
- Indexer / Proof Catalog：索引链上事件、维护 proofId ↔ block 映射，供前端与 Proof Service 查询。  
- 存储：链上仅存 merkleRoot / metadata；大资产（3D、音频、GLSL）放 S3 / IPFS；索引与用户元数据放 Postgres / Redis。  
- 前端：Three.js / Unity WebGL 场景（Archive / Now / Lab）、Wallet Connect、仪式动画、状态指示（pending / confirmed）。

---

## 安全与抗滥用策略
- 合约端验证：所有跨链交互必须在 Now 中通过 verify-spv 校验；Proof Service 仅作辅助。  
- Replay 防护：proofId 唯一、一致性检查、标记表（Pact 表）。  
- 时间窗口：验证 proofs 时限制 blockHeight 范围，防止过旧证明滥用。  
- 权限分级：Archive 写权限由 curator 多签或 DAO 管理；Lab 有隔离开发通道。  
- 审计与测试：Pact 合约静态分析、单元测试、golden tests、第三方安全审计。  
- 费控：限制 proof blob 提交大小，鼓励 batched merkleRoot 以减少 gas。

---

## 产品与用户体验（UX）策略
- 单一入口：用户只需连接 Now 链钱包，体验三场景的故事化交互。  
- 上手路径：守护神选择 + 少量空投 → 任务/探索 → Fuse 仪式（动画） → 展览/交易。  
- 隐藏复杂度：多链复杂度由 VCL / Proof Service 后台处理。  
- 社群活动：纪念展、拍卖、主题活动、艺术家合作与限时配方。  
- 创作者工具：展览套件、链上时间戳、Proof-as-a-Service API（付费/免费层次）。

---

## 治理与组织
- 初始治理：多签 curator 团队 + 项目核心团队负责早期内容与合约发布。  
- 中长期：逐步开放治理（DAO 提案）控制经济参数、合成配方、curator 权限转移。

---

## 角色分配
- Core Team：产品、后端（Haskell / Pact）、前端（3D）、DevOps、QA。  
- Curators：艺术 / 社区运营（Archive 管理）。  
- Dev Partners：Lab 孵化项目接入审核。

---

## MVP 与里程碑（0–6 个月）
### MVP（3 个月目标）
- 基础交付：
  - 3 个 Chainweb 节点（测试网）  
  - Now 链主要合约（Coin、Fuse/Claim、NFT）  
  - Archive 的纪念合约复刻部署  
  - Proof Service 基础实现、VCL Router  
  - 前端三场景最小可交互 Demo、JS SDK  
- 运维：Redis、Postgres、S3 / IPFS、Prometheus + Grafana。  
- 市场：KDA 社区 outreach、艺术家合作试点、首次纪念空投。

### 阶段化 Roadmap
- M0–M1：架构搭建、Pact 合约开发与单元测试、Proof Service PoC。  
- M1–M2：前端体验、VCL 集成、E2E 测试、curator 多签流程。  
- M2–M3：安全审计、早期社区测试、第一次纪念/融合活动。  
- M3–M6：启用 Lab 孵化通道、开放 SDK、开始 Proof-as-a-Service 商业化尝试。

---

## 运维与最小团队配置
建议核心团队（初期 6–8 人）：
- 技术负责人（Haskell / Pact）1  
- 后端开发 2（VCL、Proof Service、Indexer）  
- 前端 / 3D 场景 2（WebGL / Unity）  
- DevOps / Infra 1（k8s、监控、备份）  
- 产品 / 社区 / curator 1–2（产品、合作、艺术家联络）

---

## 指标与监控（KPI）
- 活跃用户（DAU / WAU）  
- Fuse 次数、Essence 铸造数  
- proof-gen latency、claim failure rate  
- Now 链平均确认时间、gas 消耗  
- 服务 SLO（Proof Service uptime）

---

## 风险评估与缓解
- Proof 的 gas 与存储成本过高：batch/merkleRoot、proof 压缩、off-chain pointer。  
- Proof Service 成为信任中心：链上 verify-spv 为最终正义、Proof Service 签名仅为辅助。  
- 用户等待 / 体验差：乐观反馈、明确 pending → confirmed 流程。  
- 合约漏洞：严格测试与第三方审计。  
- 运营风险（curator 滥用）：多签、透明审计与治理。

---

## 经济可持续性
收入来源：
- Fuse / Claim 手续费（微额）  
- Proof-as-a-Service 高级订阅（企业 / 艺术家）  
- 拍卖 / 展览手续费与二级市场分成  
- 企业 / 创作者托管与定制服务

---

## 合规与法律
- 项目初期以艺术 / 纪念为主，避免证券化表达；任何金融化功能（DEX、收益产品）推向合规评估与本地法律咨询。  
- 隐私：链上不存原始媒体，使用 pointers 与加密存储，遵守适用数据保护法规。

---

## 附录 A：Now 链 Fuse/Claim Pact（示意）
```lisp
;; Pact5 概念性示例（需完善与审计）
(module fuse-claim G 'admins)

(define-keyset 'admins (read-keyset "admins"))

;; table used-proofs: proofId -> {consumedBy, ts}
(defun proof-used? (proofId)
  (not (is-null (read-key "used-proofs" proofId))))

(defun mark-proof-used (proofId owner)
  (write-key "used-proofs" proofId { "consumedBy": owner, "ts": (time) })
  true)

(defun claim-from-remote (owner proof proofId meta)
  (when (proof-used? proofId) (abort "Proof already used"))
  (let ((ok? (verify-spv proof))) ;; Pact5 内置
    (when (not ok?) (abort "Invalid SPV proof"))
    ;; mint Essence NFT (abstract)
    (create-nft owner meta)
    (mark-proof-used proofId owner)
    (format "Essence minted for %s" owner)))
```

---

## 附录 B：交互时序（简述）
1. 用户在 Archive 完成“纪念铸造”。  
2. Proof Service 生成 proofId + blob。  
3. 用户在 Now 发起 claim(proofBlob)。  
4. Now 合约 verify-spv → mint Essence → mark proof used。  
5. 前端展示 confirmed。
