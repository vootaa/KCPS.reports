# Chainweb + Pact + SPV: Análise de Cenários e Design de Sistema (Português)

Versão: baseada na base de código chainweb-node-2.31.1  
Autor: GitHub Copilot (estilo consultoria de estratégia)  
Data: 2025-10-30

## 1 Resumo executivo (síntese para decisores)
Chainweb fornece infraestrutura de blockchain multi-chain paralela e altamente verificável; o suporte nativo a SPV e o sistema de contratos Pact o tornam adequado para produtos de "ancoragem de eventos verificáveis (event anchoring)", "liquidação auditável" e "validação por clientes leves".  
Para IoT (Internet das Coisas), DePIN e mercados/compute verificável de IA, recomenda-se um MVP com "Edge Gateway + API de Batch SPV + liquidação via Pact", priorizando a redução de latência na geração de provas SPV e gargalos de I/O.

## 2 Capacidades existentes (baseado em evidências do código)
- Núcleo multi-chain paralelo: src/Chainweb/* (chain, graph, gestão de versões)  
- Geração/validação de provas SPV: src/Chainweb/SPV/CreateProof.hs, src/Chainweb/SPV/VerifyProof.hs  
- Integração com Pact e interfaces REST: src/Chainweb/Pact*/RestAPI/Server.hs, exemplo PactService em bench  
- Camada de persistência e I/O: src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- Exemplo de log verificável: docs/merklelog-example.md

Esses módulos suportam diretamente o fluxo confiável “submeter resumo → inserir em lote na cadeia → gerar prova SPV → validação externa”.

### Diferenças de versão do Pact e compatibilidade
- Chainweb suporta troca entre Pact4/Pact5 (veja o tipo de dado `PactVersion` em `chainweb-node-2.31.1/src/Chainweb/Version.hs`).  
- Pact5 em `pact-5-5.4/src/Pact/Core/` otimiza cálculo de Gas e verificação SPV (`verify-spv` documentado em `pact-5-5.4/docs/builtins/SPV/verify-spv.md`), sendo mais indicado para cenários de alta vazão; é necessário garantir que os contratos especifiquem a versão correta ao serem implantados (veja `chainweb-node-2.31.1/chainweb.cabal` para `Chainweb.Pact5.SPV`).  
- Recomenda-se Pact5 para novos cenários devido às otimizações de execução concorrente (consulte o ramo Pact5 em `src/Chainweb/Pact/PactService.hs`).  
- Há testes SPV para Pact5 que podem ser reaproveitados (`chainweb-node-2.31.1/test/unit/ChainwebTests.hs` contendo `Chainweb.Test.Pact5.SPVTest.tests`).  
- Chamadas entre versões de contratos exigem tratamento em `Chainweb.Pact.Conversion` (veja referências em `chainweb-node-2.31.1/chainweb.cabal`).

Esses componentes viabilizam o fluxo completo de provas verificáveis descrito acima.

## 3 Cenários aplicáveis e proposta de valor
- IoT / DePIN: massiva submissão de eventos por dispositivos (estado, medição, evidências) com merkle root on-chain e provas SPV para auditoria ou liquidação contratual. Valor: reduzir custo de confiança e fornecer trilhas de prova imutáveis.  
- IA (computação verificável) e mercados de dados: ancorar metadados/resultados de tarefas e automatizar liquidação via Pact. Valor: precificação auditável, incentivos e responsabilidade.  
- Serviço comercial de verificação SPV (SPV-as-a-Service): fornecer API para geração e verificação de provas, reduzindo necessidade de nós completos para clientes leves. Modelo de monetização claro.  
- Jogos / GameFi: ancoragem de itens/partidas, liquidação de competições e transferência de ativos entre servidores com alta concorrência (ver 3.1).  
- Serviços de auditoria empresarial: evidências imutáveis, relatórios de conformidade automatizados e rastreabilidade na cadeia (ver 3.2).

## 3.1 Cenário de jogos (Game / GameFi)

Resumo de caso de uso
- Registro de itens e ativos on-chain: registar itens raros, NFTs e ordens como resumos na Chainweb; topologia multi-chain suporta alta concorrência.  
- Ancoragem de eventos em tempo real/próximo ao real: servidores de jogo enviam batches de eventos (snapshots de ranking, resultados) e SPV fornece provas verificáveis para mercados/árbitros externos.  
- Transferência entre servidores/mundos: usar particionamento por chain (por região/shard) e SPV/forwarder para migração de ativos cross-chain.  
- Liquidação de torneios: Pact para regras e pagamento automático; merkleRoot on-chain e Proof Service para evidências de arbitragem.

Arquitetura de alto nível (resumo)
- Servidores de jogo em tempo real: lógica de baixa latência; envios periódicos ou em lote para Edge Gateway.  
- Edge Gateway: agrega eventos, constrói MerkleLeafs, roteia por chainId (determinístico) e, se necessário, faz commits otimistas locais.  
- Anchor/Batch Ingestor: escreve merkleRoot em chain via chamada Pact ou payload write; Pact recomendado para ações que exigem liquidação.  
- Proof Service: pré-computação de provas após confirmação de bloco e cache TTL para acessos frequentes.  
- Asset Vault: contratos Pact gerenciam propriedade, transferência e arbitragem.  
- Marketplace/Wallets off-chain: solicitam provas ao Proof Service para verificar histórico do ativo.

Pontos-chave de design e trade-offs
- Latência vs finalidade: estados de jogo em tempo-real usam off-chain + ancoragem periódica; ações econômicas importantes usam on-chain Pact + SPV.  
- Particionamento multi-chain: mapear mundos/regiões para chains específicas; usar two-phase pattern para transferências cross-chain.  
- Cache e rollback: servidores de jogo suportam rollback otimista quando off-chain conflita com estado on-chain; resolução por contrato ou transação de compensação.  
- Segurança: aplicar proteção anti-abuso, verificação de assinaturas e limites SLA em Edge Gateway e Proof Service.

Exemplo de interação (resumido)
1. Jogador compra skin → GameServer envia evento para Edge Gateway.  
2. Edge Gateway faz batch e grava merkleRoot em contrato Pact recordBatch na chain indicada.  
3. Após confirmação, Proof Service pré-computa a prova; Marketplace solicita a prova e completa a transferência/exibição.

Considerações de desempenho
- Escritas concorrentes tratadas por particionamento multi-chain; se geração de provas for gargalo, usar confirmação retardada e caching.  
- Para jogos populares, distribuir múltiplas instâncias do Proof Service com Redis sharding.

## 3.2 Cenário de auditoria (Enterprise Audit / Compliance)

Resumo de caso de uso
- Provas de conformidade e auditoria: ancorar eventos críticos, faturas e logs para fornecer histórico imutável.  
- Rastreabilidade na cadeia de suprimentos: registrar etapas chave (produção, inspeção, transporte) com merkleRoot e SPV para auditores terceirizados.  
- Automatização de relatórios de conformidade: Pact para regras de notificação e disparo; provas como vouchers de auditoria.

Arquitetura de alto nível
- Data Ingestors (coleta empresarial): ETL, eventos assinados e metadados de conformidade.  
- Normalizer & Policy Engine: padroniza formato, classifica e decide política de entrada (real-time vs batch).  
- Audit Anchor Service: gera merkle batches e submete ao chainId do cliente.  
- Proof Catalog & Index: mapeia batchId → (blockHash, timestamp, proofUri, retentionPolicy) para pesquisa e arquivamento.  
- Audit Portal / Verifier: interface Web/CLI para auditores realizarem download e verificação de provas.  
- Camada de arquivamento: objeto storage para dados brutos criptografados; apenas merkleRoot permanece on-chain.

Estratégias multi-chain e isolamento de tenants
- Cada cliente pode ter partition por chainId ou contratos replicados com tenantId on-chain. Para auditoria, recomenda-se contratos replicados + registry on-chain.

Segurança e conformidade
- Cadeia de custódia: eventos assinados no ingest; operações registradas com logs de auditoria.  
- Controle de acesso: Pact + Portal off-chain implementam RBAC; divulgação mínima de dados sensíveis e estudos de ZK podem ser avaliados.  
- Retenção/Privacidade: somente merkleRoot on-chain; dados sensíveis mantidos criptografados off-chain.

APIs e modelo de dados (exemplo)
- POST /v1/audit/batch { events[], policy, tenantId } -> { batchId, submittedToChain, txHash }  
- GET /v1/audit/proof/{batchId} -> { proof, blockHash, timestamp, verifierHints }  
- GET /v1/audit/catalog?tenantId=...&from=...&to=... -> lista de batches/metadados

Operação e SLA
- SLA exemplo: disponibilidade de provas 99.9% (TTL 24h); arquivamento 7–10 anos conforme requisito regulatório.  
- Métricas: número de submissões pendentes, latência de geração de prova, tempo de acesso a cold storage, logs de auditoria.

Fluxo de auditoria (exemplo)
1. Sistema corporativo envia transações/logs assinados ao Ingestor.  
2. Anchor Service agrupa e grava batch, retornando batchId e txHash.  
3. Auditor solicita prova via Portal; Proof Service fornece; verificação local é executada.  
4. Resultado verificado armazenado em relatório de auditoria.

Extensões sugeridas
- Integração com serviço de timestamp (TSA) para reforçar rastreabilidade temporal.  
- Re-ancoragem periódica para mitigar obsolescência de hash (migração para novo hash e novo merkleRoot).

## 3.3 IoT / DePIN

Casos de uso
- Telemetria e medição em grande escala: smart meters, sensores ambientais e industriais com loteamento de eventos on-chain.  
- Autenticação de dispositivos e prova de direito de uso: registro de dispositivos, emissão de certificados e registros de uso verificáveis.  
- Micro-pagamentos dispositivo-a-dispositivo: liquidação via Pact e evidência via SPV.

Arquitetura dedicada
- SDK para dispositivos (cliente leve): assinaturas leves, empacotamento, retry/backoff e opcional construção Merkle local.  
- Edge Gateway: TLS/mTLS, controle de fluxo, deduplicação, caching edge e roteamento determinístico por chainId; batch configurável (janela temporal / capacidade).  
- Estratégia de escrita Ingest → Chain: direct payload para baixa latência; Pact para eventos críticos que exigem liquidação.  
- Proof & Verification: Proof Service com APIs para puxar/validar provas; SDKs JS/Python oferecendo verificação embutida.

Pontos de implementação
- Recomenda-se localizar Edge+Proof perto geograficamente ao cluster de dispositivos.  
- Dados sensíveis criptografados em cold storage; apenas merkleRoot on-chain.  
- Suporte a at-least-once com idempotencyKey para evitar duplicação.

Segurança e privacidade
- Gestão do ciclo de vida de identidade dos dispositivos (revogação/atualização de certificados).  
- Minimização de dados on-chain e políticas de divulgação controlada.

Métricas e tuning
- Recomenda-se janela batch 0.5–2s, tamanho 500–2000 eventos, alvo p95 ≤ 2s (MVP).

## 3.4 IA: computação verificável e marketplaces de dados

Casos de uso
- Registro de tarefas e ancoragem de resultados: treinos, inferências, metadados e resumos verificados.  
- Mercado de modelos contratualizado: escopo de trabalho, provedores enviam provas de execução e recebem pagamento via Pact.  
- Rastreabilidade de datasets: contribuintes submetem amostras com provas de originalidade e timestamp.

Arquitetura de alto nível
- Task Broker: cadastro de tarefas, matching e distribuição para compute nodes.  
- Compute Node: executa trabalhos, gera resumos assinados e envia ao Edge Gateway.  
- Result Anchoring: merkle batches e escrita em Chainweb; Pact para liquidação de alto valor.  
- Incentivos & Settlement: contratos Pact para estado de tarefa e arbitragem; Proof Service fornece cadeia de provas.

Pontos-chave
- Para outputs grandes, usar merkle de slices no compute node antes de ancorar o root.  
- Incluir logs/ambiente de execução no merkle para auditoria anti-fraude.  
- Estratégias de latência: fast-path para pequenas tarefas; final-path para tarefas críticas (força finalidade on-chain).

Governança de dados e conformidade
- Amostras cifradas/rotuladas antes de enviar; apenas índices mínimos e provas on-chain.  
- Recursos de rastreabilidade por fornecedor/tarefa.

Comercialização
- Monetização por request de prova, taxas de liquidação e serviços de arquivamento.

## 3.5 SPV-as-a-Service

Casos de uso
- API/SDK para geração/verificação de provas sob demanda para wallets, pontes cross-chain e auditorias.  
- Serviço white-label com SLA, arquivamento, RBAC e certificados de conformidade.

Arquitetura
- Public API Gateway: REST/gRPC, autenticação, cobrança por uso.  
- Proof Fabric: worker pool assíncrono, filas de prioridade, cache/índice (Redis + object storage).  
- Multi-tenant: namespaces, controle de acesso e logs de auditoria.  
- Disponibilidade: hot-standby, replicação cross-region.

Pontos principais
- Modo on-demand vs precompute: trade-off entre custo e SLA.  
- Proteções contra abuso: rate-limiting, cobrança por tamanho/complexidade.  
- Transparência: logs imutáveis de geração de provas e hashes de build.

Compliance e auditoria
- Fornecer metadados de proveniência da prova (nó gerador, versão, timestamp) e opcionalmente registrar no-chain como log de serviço.

Operação e modelo de negócios
- Precificação por request/tamanho ou assinaturas; enterprise com SLA e arquivamento.

Integração com kda-tool e deployments empresariais
- Para clientes corporativos, usar `kda-tool gen` para gerar transações multi-chain e integrar com fluxo de SPV.  
- Docker da kda-tool pode ser co-locado com chainweb-node para signing e verificação.  
- Implementar RBAC baseado em utilitários de autenticação em `Chainweb.RestAPI.Utils`.

## 4 Solução técnica (visão geral)
Objetivo: maximizar throughput sob garantia de eventual consistência, minimizando latência de verificação. Estratégia: "agregação na borda + batch on-chain + pré-computação/caching de provas".

Componentes-chave:
- Edge Gateway: recebe eventos, faz batch, valida assinaturas e aplica rate-limiting.  
- Batch SPV Ingestor: escreve merkleRoot via Pact ou payload direto.  
- SPV Proof Service: usa CreateProof, mantém cache e pré-computa provas.  
- Pact Contract Layer: contratos para registro/settlement.  
- SDKs Leves: bibliotecas de verificação para clientes (JS/Python/Go).  
- Observability: métricas para TPS/latência/I/O.

## 5 Arquitetura de alto nível (componentes e fluxo de dados)
Componentes:
1. Dispositivo / edge device: evento → assinado → Edge Gateway  
2. Edge Gateway: fila, dedup, batching, merkle root → Batch SPV Ingestor  
3. Cluster Chainweb: recebe batch via Pact/payload e gera blocos paralelos  
4. SPV Proof Service: CreateProof, cache Redis/LRU, REST API /v1/proof/{batchId}  
5. Pact contracts: gerenciam metadados e liquidações  
6. Clientes leves/verificadores: puxam prova e validam localmente

Fluxo: Dispositivo → Edge Gateway(batch) → escrita Chainweb → confirmação de bloco → Proof Service gera/cacheia prova → cliente solicita/verifica.

(Recomenda-se desenhar diagrama com boxes e camadas de armazenamento)

## 5.1 Estratégia de multi-chain e camada de abstração

Pergunta: os contratos são idênticos em cada chain? como particionar tarefas entre chains? pode ser transparente para a camada superior?

Conclusões principais:
- Contratos podem ser replicados em cada chain ou particionados por shard. Replicado simplifica, particionado escala.  
- Uso de roteamento determinístico (hash mod N) ou consistent hashing para mapear tarefas a chains.  
- Propor uma camada de abstração "Virtual Chain Layer (VCL)" que exponha Submit/Query/Call e esconda roteamento, retries, obtenção/verificação de provas.

Detalhamento estratégico
1) Modos de deployment de contrato
- Replicado: mesmo módulo Pact em todas as chains. Simples, porém exige sincronização para estado global.  
- Particionado: cada chain mantém subset do estado. Melhora throughput, mas operações cross-chain exigem protocolos adicionais.

2) Regras de roteamento
- Algoritmo recomendado: chainId = H(taskKey) mod N (usar hash consistente com a cadeia).  
- Para escala dinâmica, usar Consistent Hashing e estratégias sticky/load-aware.

3) Design da camada VCL
- Componentes: Router (decide chain), Executor (submete via REST ou payload), Verifier/Coordinator (obtem/valida provas para cross-chain).  
- APIs de exemplo:
  - submitTask(taskKey, payload, policy) -> { chainId, txHash }  
  - getResult(taskKey) -> consulta chain(s) mapeados e retorna resultado verificado  
  - callCrossChain(fromChain, toChain, payload) -> usa proof relay verify+apply

4) Padrões de chamada
A. Forwarding on-chain: deploy de forwarder em cada chain para verificar SPV on-chain e executar; auditável mas custoso em gas.  
B. Router off-chain + verify on-chain leve: valida provas off-chain e submete referências; reduz custo on-chain mas exige confiança no router.

5) Consistência cross-chain
- Padrão weak-consistency: confirmações por chain única; cross-chain eventual consistency via SPV.  
- Padrão strong-consistency (quando necessário): two-phase commit + prova SPV para commit; caro em latência.

6) Deploy e sincronização de contratos
- Automatizar deploy com scripts CI/CD e manter registry on-chain com hashes/versões de contratos.

7) Localização do novo código
- Novo módulo sugerido: src/Chainweb/Multichain/Router.hs ou extensão no Pact REST Server.  
- Reaproveitar CreateProof/VerifyProof, Pact REST Server e PayloadStore/RocksDB.  
- MVP: implementar roteamento determinístico e Proof Service off-chain para relay.

7) Riscos e trade-offs
- Provas cross-chain aumentam latência e custo; preferir partitioning para baixa latência.  
- Contratos replicados facilitam rollout porém dispersam estado.  
- Off-chain Router precisa de mecanismos de descentralização/arbítrio para reduzir confiança.

Exemplo em pseudocódigo (resumido)
```pseudo
// submitTask(taskKey, payload, policy)
chainCount = getActiveChainCount()
chainId = H(taskKey) % chainCount
if policy == "replicated":
  for c in allChains:
    submitToChain(c, payload)
  return aggregateTxs()
else:
  tx = submitToChain(chainId, payload)
  return { chainId, txHash: tx.hash }
```

Cross-chain via proof relay:
```pseudo
// callCrossChain(fromChain, toChain, txHash, callPayload)
proof = CreateProof(fromChain, txHash)
verifyLocally = VerifyProof(proof)
if verifyLocally:
  submitToChainWithProof(toChain, callPayload, proof)
else:
  fail("proof invalid")
```

Prioridades de engenharia:
1. Deterministic routing + Edge Gateway retorna chainId (MVP).  
2. Off-chain proof relay (Proof Service) exposto na VCL.  
3. Two-phase pattern para cenários que exigem atomicidade cross-chain.

## 6 Recomendações técnicas detalhadas (por módulo)
1. Edge Gateway
- API: POST /v1/events/batch { events[], batchSize, ttl } -> batchId  
- Estratégia de batch: janela temporal (ex. 1s) ou capacidade (ex. 1000 eventos)  
- Construir Merkle usando mesmo hash da cadeia (ver Version.hs)  
- Validar assinaturas e capturar metadata do submitter

2. Escrita em lote para chain
- Dois caminhos:
  a) Pact call: grava metadata em contrato (bom para liquidação)  
  b) Direct payload write: menor latência, mas exige metadado off-chain para liquidação  
- Recomenda-se Pact para casos que exigem liquidação contratual

3. Otimizações no Proof Service
- Pré-cálculo/geração incremental de provas assim que blocos forem minerados  
- Cache em Redis para provas quentes, snapshot em disco para provas frias  
- Pool de workers com limite de concorrência para evitar conflitos RocksDB

4. Cache e index
- Manter map batchId → (blockHash, leafIndex, merklePathCached)  
- Usar memória mapeada e Redis para indexação rápida

5. Protocolos e formatos de dados
- Batch meta: { batchId, merkleRoot, submitter, timestamp, chainId, blockHeight, proofUri }  
- Formato de prova: saída de VerifyProof.hs com serialização binária e JSON  
- Endpoints: GET /v1/proof/{batchId}, POST /v1/proof/verify

6. Segurança e consistência
- Nonce/time-stamp anti-replay entre submitter e nó  
- Garantir compatibilidade entre algoritmos de hash (versão em Version.hs)  
- Registrar logs de auditoria on/off-chain

Segurança pós-quantum
- Atualmente uso de SHA512t_256; documento recomenda considerar migração para Keccak-256 conforme `src/Chainweb/SPV/PayloadProof.hs` (opção SpvKeccak_256).  
- Versionamento do algoritmo de hash em `src/Chainweb/Version.hs`.  
- Reforçar prevenção contra replay em `Chainweb.RestAPI.Utils` e validação de assinatura em contratos Pact.

## 7 Operações e desempenho
- Gargalos principais: I/O do RocksDB (compaction/latência) e CPU para construção de provas Merkle.  
- Métricas recomendadas: p50/p95/p99 end-to-end (entrada → prova disponível), QPS de proof, latência média de geração, IOPS RocksDB.  
- Deployment: Edge Gateways próximos dos dispositivos; Proof Service co-locado com Chainweb nodes ou em instâncias dedicadas; Redis distribuído.

Testes e benchmarking
- Reaproveitar testes existentes: `cabal test` para SPV, Pact SPV integration tests (veja `test/unit`).  
- Benchmarks sugeridos com utilitários em test/lib e pact-repl para medir latência de execução Pact e geração de provas.  
- Incluir testes end-to-end para roteamento multi-chain.

Modelo matemático de desempenho (resumido)
- Throughput aproximado: T ≈ (n · t_c) / (1 + B + C), onde C é fator de hit de cache.  
- Monitorar IOPS e ajustar RocksDB via parâmetros em `src/Chainweb/Payload/PayloadStore/RocksDB.hs`.

## 8 Sugestões de mudança de código (prioridade)
Alta prioridade (MVP):
- Em src/Chainweb/SPV/CreateProof.hs: adicionar API assíncrona de precompute e interface de cache (Redis).  
- Em src/Chainweb/Pact/RestAPI/Server.hs: adicionar endpoints para batch write e consulta por batchId (ou criar novo Edge Gateway service).  
- Em src/Chainweb/Payload/PayloadStore/RocksDB.hs: otimizar caminho de escrita em lote e expor parâmetros de tunning.

Médio prazo:
- Implementar Proof Service com camadas Redis e LRU, reutilizando template PactService em bench.  
- Publicar SDKs para JS/Python e exemplos.

Longo prazo:
- Explorar aceleração de hardware para hashing e provas, e integração com ZK.

## 9 Roadmap sugerido
- Mês 0–1: definir API do Edge Gateway, PoC device→gateway→chain.  
- Mês 1–3: implementar fluxo Batch SPV + Proof Service com cache (MVP).  
- Mês 3–6: otimizações (RocksDB, worker pool), publicar SDK, PoC clientes.  
- Mês 6–12: features enterprise (RBAC, monitoria), explorar aceleração por hardware.

## 10 Riscos, custos e mitigação
- Riscos: I/O bottleneck, latência de geração de provas, complexidade de contratos.  
- Mitigações: batch, precompute, caching e arquitetura em camadas (payload vs pact).

## 11 Conclusão
Aproximação recomendada: "Edge Gateway + Batch SPV + Pact para liquidação". MVP viável em ~3 meses; após isso, escalar via pré-cálculo de provas e cache para uso em IoT/DePIN/mercados de IA.

## 12 Referências de código (para engenharia)
- Geração de provas: src/Chainweb/SPV/CreateProof.hs  
- Verificação de provas: src/Chainweb/SPV/VerifyProof.hs  
- Pact REST Server: src/Chainweb/Pact*/RestAPI/Server.hs  
- Armazenamento Payload (RocksDB): src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- Exemplo PactService: bench/Chainweb/Pact/Backend/PactService.hs  
- Exemplo MerkleLog: docs/merklelog-example.md  
- Versionamento: src/Chainweb/Version.hs  
- Relatório consolidado: [`ChainwebPactSPV_AvaliacaoEstrategicaRecomendacoes_pt.md`](./ChainwebPactSPV_AvaliacaoEstrategicaRecomendacoes_pt.md) 
- Ecosistema: mencionar chainweb-mining-client-0.7 como ferramenta de minerador aplicável a DePIN.
