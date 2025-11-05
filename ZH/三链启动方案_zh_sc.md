# 三链启动方案（简短）

- 三个方案对比：20 链＝最大吞吐 / 复杂度最高；10 链＝折中；3 链＝最低工程/运营成本、最佳首发用户体验。  
- 建议首发采用 3 链架构以快速获得用户与客户；设计时保留可扩展接口与抽象层，后续逐步扩容到 10/20 链。

## 一、三种方案快速对比

| 方案 | 吞吐 / 复杂度 | 主要优势 | 主要风险 | 适用场景 |
|---|---:|---|---|---|
| 3 链（两两相连） | ≈3× 单链，复杂度最低 | 最小开发/运维成本，快速上线与测试；“单链感知”UX；适合 MVP | 吞吐上限低，跨链能力有限；需合适的证明/聚合层保留未来扩展 | 快速验证市场、企业/行业落地（IoT、审计、DePIN）、MVP |
| 10 链（PetersenGraph） | 较高吞吐，复杂度中等 | 吞吐与容错折中；比 20 链更易运维 | 仍需复杂路由/运维与聚合层 | 中型项目、有扩展计划且有更多工程/资金支持 |
| 20 链（并行） | 理论吞吐最高，复杂度最高 | 最大吞吐、与 Chainweb 原生设计一致（完整 Cut/邻链引用） | 开发/运维成本高，跨链/同步复杂，UX 分散 | 需要大规模吞吐并具备长期社区/节点运营能力的主网 |

### 20 条并行链
- 优势：理论吞吐最高，原生与 Chainweb 设计一致。  
- 风险：开发/运维成本高，同步与跨链复杂，用户体验分散。  
- 适用场景：大规模吞吐、稳定长期社区与节点运营的主网。

### 10 条（PetersenGraph）
- 优势：保留较好吞吐与容错，工程折中。  
- 风险：运维与路由仍复杂，需聚合层优化 UX。  
- 适用场景：中型项目，准备扩展且有更多工程/资金支持。

### 3 条（最简化、两两相连）
- 优势：最低开发/运维成本，易测试与上线；可实现“单链感知” UX。  
- 风险：吞吐提升有限（≈3× 单链）；需设计证明/聚合层以保留扩展能力。  
- 适用场景：快速验证市场、企业/行业落地（审计、IoT、Proof-as-a-Service）、MVP 首选。

## 二、若选择 3 链 — 技术架构（核心组件）

- Edge Gateway（边缘聚合）  
    - 收集事件/交易，构建 Merkle-batch，做 rate-limit / signature check，决定 routing key。  
    - 输出 batchId + merkleRoot → 交由 VCL 提交。

- Virtual Chain Layer (VCL) / Router  
    - 将上层提交映射到具体 chainId（deterministic hash mod 3，或 tenant→sticky mapping）。  
    - 对外提供单一 API（submit / query / verify），隐藏多链细节。

- Chainweb 节点群（3 个并行链，保留 Cut 机制）  
    - 使用 Chainweb Node 精简配置（3 节点全连图），PactService 优先 Pact5（性能与 SPV 支持更好）。

- Pact 合约层  
    - 部署通用合约模板（batch registry、escrow/settlement、access-control）。优先 Pact5，保留 Pact4 兼容路由。

- SPV Proof Service（Proof Fabric）  
    - 异步 precompute + Redis 缓存 CreateProof/VerifyProof；提供 REST API，支持优先队列收费策略。

- Proof Catalog / Indexer / Aggregator  
    - 收集区块/tx → 生成索引 → 维持 batchId → (blockHash, leafIndex, proofUri)，对外提供统一视图。

- PayloadStore / PactState 后端  
    - RocksDB + SQLite v2（Pact state），小规模实例并启用 WAL / compaction tuning。

- API Gateway / SDK / Explorer / Wallet  
    - 单一登录与钱包 UX，Explorer 聚合多链视图并在后台拼接 SPV 验证结果。

- Observability & Ops  
    - 指标：proof-gen latency、proof QPS、mempool depth、block lag per chain、RocksDB IOPS；日志与告警。

## 三、数据流（简短）

客户/设备 → Edge Gateway（batch + merkleRoot）  
→ VCL router（选择 chain）→ Chain 提交（Pact call 或 payload write）  
→ Chain 确认 → Indexer 收到 block → Proof Service 预计算并缓存 proof  
客户请求 proof → Proof Service 返回 proof/VerifyProof API  
VCL 聚合多链数据并对外呈现“单链体验”

## 四、设计细节与工程要点

- 抽象 ChainGraph/Guards：在 Graph.hs / Version.Guards 中配置 3-node graph，并实现运行时可扩展的参数化 graph。  
- Pact 版本：首发强推 Pact5（并发/Gas/SPV 更优），保留 Pact4 fallback。  
- SPV 优化：预计算（block 挖出即触发 worker）、Redis 缓存、收费优先队列（SLA）。  
- UX 抽象：所有用户与 VCL 交互（单一 RPC endpoint + SDK），内部负责路由/重试/跨链 proof 聚合。  
- 多租户：支持 namespace per tenant 或每租户固定 chain mapping，便于隔离与 SLO。  
- 存储策略：链上仅存 merkleRoot/metadata，原始数据存外部冷库（S3）。  
- 测试：保留 golden tests（test/golden），扩展 E2E tests for proof flow。

## 五、产品策略（产品专家侧）

- 核心定位：SPV-as-a-Service + Pact-based settlement for industry（IoT / Audit / DePIN）。  
- MVP（最小可行产品）  
    - API: submitBatch -> { batchId }、getProof(batchId)、verifyProof  
    - Pact 模板：recordBatch、escrow/settlement、tenantRegistry  
    - Simple dashboard：batches、proof status、metrics  
    - JS/Python SDK + Postman collection

- 用户体验原则  
    - 隐藏多链复杂性（VCL single endpoint）  
    - 快速反馈：Edge Gateway 乐观 ack；finality UI 显示 proof 就绪  
    - 简化 onboarding：CLI + dockerized node + hosted Proof Service

- 商业化与定价  
    - Free tier（开发者），pay-per-proof（生产），subscription（SLA & 长期归档）  
    - Enterprise node + managed service + consulting

- Go-to-market  
    - 目标行业：IoT/DePIN、审计公司、游戏工作室（proof-of-result）、供应链 SaaS  
    - Demo kits、SDK、示例 Pact 模板、quick-start terraform/docker

## 六、最小团队与时间线（参考）

- Team (core): 1 tech lead、2 backend devs（Haskell/Pact）、1 infra/devops、1 product/PM、1 SRE/QA、1 bizdev（6–7 人）  
- Timeline（MVP ~ 3 个月）  
    - Month 0：设计、基础 infra（k8s、Redis、S3）、3-node graph config  
    - Month 1：Edge Gateway + VCL + 基础 Chainweb 部署 + Pact5 模板  
    - Month 2：Proof Service（precompute + cache）+ SDKs + 基础 dashboard  
    - Month 3：E2E tests、security audit、早期客户 pilot

- Ops footprint：3 个 chainweb 节点、2 个 Proof Service 副本、Redis 集群、Postgres/indexer、S3、监控（Prometheus+Grafana）

## 七、扩展路径到 10/20 链（演进策略）

- 设计原则：向后兼容、逐步扩容、可热插拔 chain graph。  
- 必要改动：Graph/Version 参数化（动态加载）、VCL 路由改为一致性哈希、Contract deployment automation（CI）、Reindexer（state snapshot + 验证）  
- 演进步骤：启用更多链配置并启动新节点 → 部署合约副本或分区合约 → 运行集成测试与 golden checks → 通过 VCL 渐进流量切换

## 八、主要风险与缓解

- SPV 生成延迟：预计算、缓存、优先队列。  
- 用户感知多链复杂：VCL 隐藏、统一 Explorer。  
- 存储膨胀：链上仅存 merkleRoot，原始数据外部冷存。  
- 跨链原子性复杂：首发避免复杂跨链，使用 two-phase / off-chain coordinator + on-chain verifySPV。

## 九、可交付的首批技术产物（建议）

- VCL Router 服务 + API spec  
- Edge Gateway reference impl（Docker）  
- 3-chain Chainweb node config + Pact5 sample contracts  
- SPV Proof Service（Redis cache + REST API）  
- JS/Python SDK + demo app（game/audit）  
- CI E2E tests & golden hashes

## 十、下一步（操作性）

- 决策：确认首发采用 3 链并批准 MVP 资源（team + infra budget）。  
- 建立 repo/module：新增 `src/Chainweb/Multichain/VCLRouter.hs`（或在 node 的 REST 层暴露），实现 deterministic routing。  
- 优先任务：实现 Proof Service precompute + Redis cache，以及 Edge Gateway batch submit API。  
- 开始 pilot：找 1–2 家行业客户（IoT/审计/游戏）做 3 个月试点。

总结一句话：为快速被采用而活下来，首发用 3 链（隐藏多链复杂性、以 SPV 服务和 Pact 模板驱动产品化），架构从一开始设计为可模块化扩展到 10/20 链，以便在获得用户与运维经验后安全放大吞吐与网络规模。