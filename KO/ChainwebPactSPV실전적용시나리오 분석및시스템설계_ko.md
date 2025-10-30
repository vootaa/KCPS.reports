# Chainweb + Pact + SPV 실전 적용 시나리오 분석 및 시스템 설계 (한국어)

버전: 코드베이스 chainweb-node-2.31.1 기반  
작성자: GitHub Copilot (전략 기술 자문 스타일)  
날짜: 2025-10-30

## 1 요약 (의사결정자용)
Chainweb은 병렬 다중체인과 높은 검증 가능성을 제공하는 블록체인 인프라입니다. 네이티브 SPV 지원과 Pact 계약 시스템은 "검증 가능한 이벤트 앵커링(event anchoring)", "감사 가능한 정산", "라이트 클라이언트 검증" 제품군에 적합합니다. IoT, DePIN, AI 검증 계산/데이터 마켓을 목표로 할 경우 MVP로는 "Edge Gateway + Batch SPV API + Pact 계약 정산"을 권장하며, SPV 증명 생성 지연과 I/O 병목을 우선 해결해야 합니다.

## 2 기존 역량 (소스 코드 기반 증거)
- 병렬 다중체인 코어: src/Chainweb/* (chain, graph, 버전 관리)  
- SPV 증명 생성/검증: src/Chainweb/SPV/CreateProof.hs, src/Chainweb/SPV/VerifyProof.hs  
- Pact 통합 및 REST 인터페이스: src/Chainweb/Pact*/RestAPI/Server.hs, Bench의 PactService 예제  
- 영구 저장소 및 I/O: src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- 검증 가능한 로그 예시: docs/merklelog-example.md

이 모듈들은 "요약 전송 → 배치로 체인에 입력 → SPV 증명 생성 → 외부 검증"의 엔드투엔드 신뢰 흐름을 직접 지원합니다.

### Pact 버전 차이 및 호환성
- Chainweb은 Pact4/Pact5 전환을 지원합니다 (참조: chainweb-node-2.31.1/src/Chainweb/Version.hs 의 `PactVersion` 타입).  
- Pact5는 `pact-5-5.4/src/Pact/Core/`에서 Gas 계산 및 SPV 검증(`verify-spv`)을 최적화하여 높은 처리량 시나리오에 적합합니다. 배포 시 계약이 버전을 명시해야 합니다 (참조: chainweb-node-2.31.1/chainweb.cabal 의 `Chainweb.Pact5.SPV` 모듈).  
- 신규 시나리오에는 병렬 실행 최적화 때문에 Pact5를 권장합니다. Pact5에 대한 SPV 테스트들이 존재하므로 재사용 가능합니다 (체크: chainweb-node-2.31.1/test/unit/ChainwebTests.hs 의 `Chainweb.Test.Pact5.SPVTest.tests`).  
- 버전 간 계약 호출은 `Chainweb.Pact.Conversion`에서 호환성 처리가 필요합니다.

이들 구성 요소는 앞서 설명한 증명 가능한 흐름을 실현합니다.

## 3 적용 가능한 시나리오 및 가치 제안
- IoT / DePIN: 대량의 디바이스 이벤트(상태, 계측, 증거)를 Merkle 루트로 온체인 앵커링하고 SPV 증명으로 감사 또는 계약 정산을 수행. 가치: 신뢰 비용 축소 및 불변 증거 제공.  
- AI 검증 계산 및 데이터 마켓: 작업 메타데이터/결과 요약을 앵커하고 Pact로 자동 정산. 가치: 감사 가능 가격 책정, 인센티브 및 책임 규정.  
- SPV-as-a-Service: 증명 생성/검증 API 제공으로 라이트 클라이언트의 전체 노드 의존도 감소. 수익화 모델 명확.  
- 게임 / GameFi: 아이템/매치 앵커링, 대회 정산, 서버 간 자산 이동의 고동시성 처리(3.1 참조).  
- 엔터프라이즈 감사: 불변 증거, 컴플라이언스 보고 자동화 및 공급망 추적(3.2 참조).

## 3.1 게임 시나리오 (Game / GameFi)

사례 요약
- 온체인 아이템 및 자산 등록: 희귀 아이템, NFT, 주문 등을 체인에 요약으로 기록. 다중체인 병렬 처리로 높은 동시성 지원.  
- 실시간/근실시간 이벤트 앵커링: 게임 서버가 이벤트를 배치 제출(랭킹 스냅샷, 경기 결과 등)하고 SPV가 외부 마켓/중재자에 검증 가능한 증거 제공.  
- 서버/월드 간 자산 이동: 지역/월드별로 체인 파티셔닝을 사용하고 SPV/포워더로 크로스체인 이전 및 증명 검증.  
- 토너먼트 정산: Pact 계약으로 규칙과 자동 지급을 구현; 핵심 결과는 MerkleRoot로 체인에 기록하고 Proof Service가 중재용 증거 제공.

아키텍처 요약
- 게임 서버(실시간): 낮은 지연의 게임 로직, 주기적 또는 배치로 Edge Gateway에 전송.  
- Edge Gateway: 이벤트 집계, Merkle 리프 생성, chainId(결정적)로 라우팅, 필요시 로컬 낙관적 커밋 지원.  
- Anchor/Batch Ingestor: Pact 호출 또는 페이로드 쓰기를 통해 지정 체인에 MerkleRoot 기록. 금융적 결제가 필요한 경우 Pact 권장.  
- Proof Service: 블록 확정 후 증명 선계산 및 TTL 캐시를 제공.  
- Asset Vault: 소유권, 이전, 중재를 Pact 계약으로 관리.  
- 오프체인 마켓/지갑: Proof Service에서 증명 요청하여 자산 이력 검증 후 거래 또는 전시.

설계 포인트 및 트레이드오프
- 지연 vs 최종성: 실시간 게임 상태는 오프체인 + 주기적 앵커링; 중요 경제 이벤트는 온체인 Pact + SPV.  
- 파티셔닝: 월드/지역을 체인에 매핑; 크로스체인 이전은 two-phase 패턴 권장.  
- 캐시 및 롤백: 게임 서버는 오프체인 충돌 시 낙관적 롤백 지원; 계약 또는 보상 트랜잭션으로 해결.  
- 보안: Edge Gateway 및 Proof Service에 대해 악용 방지, 서명 검증, SLA 제한 적용.

상호작용 예시 (요약)
1. 플레이어가 스킨 구매 → GameServer가 Edge Gateway에 이벤트 전송  
2. Edge Gateway가 배치 처리 후 지정 체인의 Pact recordBatch에 MerkleRoot 기록  
3. 확정 후 Proof Service가 증명 사전 계산; Marketplace가 증명을 받아 표시/전환 완료

성능 고려
- 동시 쓰기는 다중체인 파티셔닝으로 처리; 증명 생성이 병목이면 지연 확인과 캐시 사용.  
- 인기 게임에는 여러 Proof Service 인스턴스와 Redis 샤딩 권장.

## 3.2 감사 시나리오 (Enterprise Audit / Compliance)

사례 요약
- 컴플라이언스/감사용 증거: 중요 이벤트, 청구서, 로그를 앵커링하여 불변의 증거 제공.  
- 공급망 추적: 생산/검사/운송 단계의 핵심 이벤트를 MerkleRoot로 기록하고 제3자 감사가 SPV로 검증.  
- 컴플라이언스 보고 자동화: Pact로 규칙/트리거 처리; 증명은 감사 증빙으로 보관.

아키텍처 요약
- Data Ingestors(수집): ETL, 서명된 이벤트, 컴플라이언스 메타데이터.  
- Normalizer & Policy Engine: 형식 표준화, 분류, 입체 결정(실시간 vs 배치).  
- Audit Anchor Service: Merkle 배치 생성 및 고객 지정 chain에 제출.  
- Proof Catalog & Index: batchId → (blockHash, timestamp, proofUri, retentionPolicy) 매핑 저장.  
- Audit Portal / Verifier: 웹/CLI로 감사인이 증명 다운로드 및 검증.  
- 아카이빙: 원본 데이터는 암호화된 오브젝트 스토리지에 보관; 체인에는 MerkleRoot만 유지.

멀티체인 전략 및 테넌트 격리
- 고객별로 chainId 파티셔닝 또는 복제된 계약에 tenantId를 두는 방식. 감사용으로는 계약 복제 + 온체인 레지스트리 권장.

보안 및 컴플라이언스
- 증거 체인: 입력 시 이벤트 서명; 모든 작업에 감사 로그.  
- 접근 제어: Pact + 오프체인 포털로 RBAC 구현; 민감 데이터 최소 공개 및 ZK 연구 가능.  
- 보관/프라이버시: 온체인에는 MerkleRoot만; 민감 데이터는 암호화된 오프체인 저장.

API 및 데이터 모델 (예시)
- POST /v1/audit/batch { events[], policy, tenantId } -> { batchId, submittedToChain, txHash }  
- GET /v1/audit/proof/{batchId} -> { proof, blockHash, timestamp, verifierHints }  
- GET /v1/audit/catalog?tenantId=...&from=...&to=... -> 배치/메타 목록

운영 및 SLA
- 예시 SLA: 증명 가용성 99.9% (TTL 24h); 장기 보관 7–10년(규제 요건에 따름).  
- 지표: 대기 중 제출 수, 증명 생성 지연, 콜드 스토리지 접근 시간, 감사 로그.

감사 흐름 예시
1. 기업 시스템이 서명된 로그를 Ingestor로 전송  
2. Anchor Service가 배치로 체인에 기록하고 batchId, txHash 반환  
3. 감사관이 Portal에서 proof 요청 → Proof Service 제공 → 로컬에서 VerifyProof 실행  
4. 검증 결과를 감사 리포트에 저장

확장 제안
- TSA(타임스탬프 인증 서비스) 통합으로 시간 추적 강화.  
- 해시 알고리즘 위험 완화를 위한 주기적 재앵커링(해시 마이그레이션).

## 3.3 IoT / DePIN

사례
- 대규모 텔레메트리/계측: 스마트 미터, 환경/산업 센서 이벤트 배치 앵커링.  
- 디바이스 인증 및 사용 권한 증명: 디바이스 등록, 증명서 발급, 사용 이력 검증.  
- 디바이스 간 마이크로페이먼트: Pact로 정산하고 SPV로 지불 증거 제공.

전용 아키텍처
- 디바이스 SDK(경량): 서명, 패키징, 재시도/백오프, 선택적 로컬 Merkle 구성.  
- Edge Gateway: TLS/mTLS, 흐름 제어, 중복 제거, 엣지 캐싱, 결정적 chainId 라우팅; 배치 윈도우/용량 설정 가능.  
- 쓰기 전략: 지연 민감성에 따라 direct payload(저지연) 또는 Pact(중요 이벤트) 선택.  
- Proof & Verification: Proof Service API 제공; JS/Python SDK에 검증 내장.

구현 포인트
- Edge+Proof를 디바이스에 지리적으로 가깝게 배치 권장.  
- 민감 데이터는 암호화된 콜드스토리지에 보관, 체인에는 MerkleRoot만.  
- at-least-once 보장과 idempotencyKey로 중복 방지.

보안 및 프라이버시
- 디바이스 ID 수명 관리(폐기/업데이트).  
- 온체인에는 최소 요약만, 민감 데이터는 오프체인 관리.

지표 및 튜닝
- 권장 배치 윈도우 0.5–2s, 배치 크기 500–2000 이벤트, 목표 p95 ≤ 2s (MVP).

## 3.4 AI: 검증 가능한 계산 및 데이터 마켓

사례
- 작업 등록 및 결과 앵커링: 학습/추론 결과, 메타데이터, 요약의 온체인 보관.  
- 계약화된 모델 마켓플레이스: 작업 범위, 제공자는 실행 증명 제출 후 Pact로 보상 수령.  
- 데이터셋 출처 추적: 기여자가 샘플 제출 및 원본성/타임스탬프 증명 제공.

아키텍처 요약
- Task Broker: 작업 등록, 매칭, 컴퓨트 노드 분배.  
- Compute Node: 작업 수행, 요약 생성 및 서명, Edge Gateway로 전송.  
- Result Anchoring: Merkle 배치 생성 후 Chainweb에 기록; 고가치 작업은 Pact로 정산.  
- Incentives & Settlement: Pact가 작업 상태 및 중재 로직 관리; Proof Service는 증명 체인 제공.

핵심 포인트
- 큰 출력은 Compute Node에서 슬라이스 Merkle을 생성하고 root만 앵커링.  
- 실행 로그/환경을 Merkle에 포함하여 부정행위 대응 가능.  
- 레이턴시 전략: 작은 작업은 fast-path, 중요 작업은 final-path(온체인 확정).

데이터 거버넌스
- 샘플 암호화/라벨링 후 전송; 체인에는 최소 인덱스와 증명만 보관.  
- 공급자/작업 단위의 추적 가능성 제공.

상업화
- 증명 요청 과금, 정산 수수료, 보관 서비스 등으로 수익화 가능.

## 3.5 SPV-as-a-Service

사례
- 지갑, 교량, 감사용으로 온디맨드 증명 생성/검증 API 제공.  
- SLA, 장기 보관, RBAC, 규정 준수 증명 포함한 화이트라벨 서비스.

아키텍처
- 퍼블릭 API 게이트웨이: REST/gRPC, 인증, 과금.  
- Proof Fabric: 워커 풀, 우선순위 큐, Redis + 오브젝트 스토리지 캐시/인덱스.  
- 멀티테넌시: 네임스페이스, 접근 제어, 감사 로그.  
- 가용성: 핫스탠바이, 리전 간 복제.

핵심 고려사항
- 온디맨드 vs 프리컴퓨트: 비용과 SLA의 트레이드오프.  
- 남용 방지: 레이트 리밋, 사이즈/복잡도 기반 과금.  
- 투명성: 생성 로그, 런타임/빌드 해시 제공.

컴플라이언스 및 감사
- 증명 생성 메타데이터(생성 노드, 버전, 타임스탬프) 제공; 필요 시 서비스 로그를 온체인에 기록.

운영 및 비즈니스 모델
- 요청별/크기별/구독형 가격; 엔터프라이즈용 SLA와 아카이빙 옵션 제공.

kda-tool 통합 및 엔터프라이즈 배포
- 기업용으로 `kda-tool gen`을 사용하여 멀티체인 트랜잭션 생성하고 Chainweb SPV 워크플로에 통합.  
- kda-tool Docker 이미지를 chainweb-node와 같은 환경에 배치해 서명 및 검증 CLI 제공.  
- Chainweb.RestAPI.Utils 기반 인증을 통해 RBAC 구현 가능.

## 4 기술적 솔루션 (개요)
목표: 최종적 일관성 보장을 유지하면서 처리량을 극대화하고 검증 지연을 최소화. 전략: "엣지 집계 + 배치 온체인 + 증명 선계산/캐시".

핵심 컴포넌트:
- Edge Gateway: 이벤트 수신, 배치, 서명 검증, 레이트제어.  
- Batch SPV Ingestor: MerkleRoot를 Pact 또는 직접 페이로드로 Chainweb에 기록.  
- SPV Proof Service: CreateProof를 사용하여 증명 생성 및 캐시 제공.  
- Pact Contract Layer: 등록/정산을 위한 계약 템플릿.  
- 경량 클라이언트 SDK: 증명 검증 라이브러리(JS/Python/Go).  
- 가시성/모니터링: TPS/레턴시/I/O 메트릭.

## 5 시스템 개요 아키텍처 (구성요소 및 데이터 흐름)
구성요소:
1. 디바이스/엣지 디바이스: 이벤트 → 서명 → Edge Gateway  
2. Edge Gateway: 큐, 중복 제거, 배치, Merkle 루트 생성 → Batch SPV Ingestor에 전송  
3. Chainweb 노드: 배치 수신 및 블록 생성(병렬 체인)  
4. SPV Proof Service: CreateProof 실행, Redis/LRU 캐시, REST API /v1/proof/{batchId} 제공  
5. Pact 계약: 메타데이터 및 정산 관리  
6. 라이트 클라이언트/검증자: 증명 요청 및 로컬 검증

데이터 흐름: 디바이스 → Edge Gateway(batch) → Chainweb 쓰기 → 블록 확정 → Proof Service가 증명 생성/캐시 → 클라이언트가 요청/검증

(다이어그램 권장)

## 5.1 멀티체인 전략 및 추상화 레이어

질문: 계약은 각 체인에 동일하게 배포해야 하는가? 태스크를 어떻게 분배할 것인가? 상위 레이어에 투명하게 추상화 가능한가?

핵심 결론:
- 계약은 각 체인에 복제(deployed replicated)하거나 파티셔닝(partitioned)할 수 있음. 복제는 단순성, 파티셔닝은 확장성에 유리.  
- 라우팅은 결정적 해시(H(taskKey) mod N) 또는 일관성 해시를 사용 권장.  
- "Virtual Chain Layer(VCL)"라는 추상화 계층을 제안: Submit/Query/Call API를 노출하고 라우팅/리트라이/증명 검증을 내부에서 처리.

전략 상세
1) 배포 모드
- 복제 배포(Replicated): 동일 Pact 모듈을 모든 체인에 배포. 단순하지만 전역 상태 동기화 필요 시 추가 작업 필요.  
- 파티셔닝(Partitioned): 각 체인이 데이터의 일부만 책임. 높은 처리량 가능하지만 크로스체인 작업 복잡.

2) 라우팅 규칙
- 권장: chainId = H(taskKey) mod N (체인의 해시 버전과 일치하도록).  
- 동적 확장에는 Consistent Hashing 추천.

3) VCL 설계
- 구성: Router(체인 결정), Executor(제출), Verifier/Coordinator(SPV 증명 획득 및 검증).  
- 예시 API:
  - submitTask(taskKey, payload, policy) -> { chainId, txHash }  
  - getResult(taskKey) -> 매핑된 체인 조회 후 검증된 결과 반환  
  - callCrossChain(fromChain, toChain, payload) -> proof relay 및 verify+apply 사용

4) 호출 패턴
A. 온체인 포워딩: 각 체인에 포워더 계약 배포하여 SPV를 검증하고 실행 — 감사 가능하나 가스비 및 비용 높음.  
B. 오프체인 라우터 + 온체인 경량 검증: 오프체인에서 증명 검증 후 참조만 제출하여 비용 절감 — 라우터에 대한 신뢰 요구.

5) 크로스체인 일관성
- 약한 일관성 패턴: 단일 체인 확정 후 SPV로 점진적 일관성 확보.  
- 강한 일관성: 2PC + SPV 기반 커밋 (지연과 비용이 높음).

6) 배포 및 동기화
- CI/CD로 Pact 배포 자동화, 온체인 레지스트리에 버전/해시 기록.

7) 코드 위치 제안
- 신규 모듈: src/Chainweb/Multichain/Router.hs 또는 Pact REST Server 확장.  
- 재사용: CreateProof/VerifyProof, Pact REST Server, PayloadStore/RocksDB.

MVP 제안
1. 결정적 라우팅(hash mod N) 및 Edge Gateway가 chainId 반환 (MVP)  
2. Edge Gateway/Router에서 단일 체인 제출 래퍼 구현  
3. 오프체인 aggregator를 통한 크로스체인 읽기 및 proof relay 구현

리스크/트레이드오프
- 크로스체인은 지연과 비용 증가; 저지연에는 파티셔닝 추천.  
- 복제는 롤아웃이 쉬우나 상태 분산 문제 발생.  
- 오프체인 라우터의 신뢰 경계를 낮추기 위해 다중 인스턴스/감사 로그 설계 필요.

의사결정용 의사코드(요약)
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

크로스체인 proof relay:
```pseudo
// callCrossChain(fromChain, toChain, txHash, callPayload)
proof = CreateProof(fromChain, txHash)
verifyLocally = VerifyProof(proof)
if verifyLocally:
  submitToChainWithProof(toChain, callPayload, proof)
else:
  fail("proof invalid")
```

우선순위
1. 결정적 라우팅 + Edge Gateway chainId 응답 (MVP)  
2. 오프체인 proof relay (Proof Service) 및 VCL API 공개  
3. 필요 시 two-phase 패턴 도입

## 6 모듈별 세부 구현 권고
1. Edge Gateway
- API: POST /v1/events/batch { events[], batchSize, ttl } → batchId  
- 배치 전략: 시간창(예: 1s) 또는 용량(예: 1000 이벤트)  
- Merkle은 체인과 동일한 해시 함수 사용(Version.hs 참조)  
- 서명 검증 및 제출자 메타데이터 기록

2. 체인 배치 쓰기
- 경로:
  a) Pact 호출: 메타데이터를 계약에 저장(정산 편리)  
  b) 직접 payload write: 낮은 지연, 정산은 별도 메타 관리 필요  
- 권장: 정산이 필요한 경우 Pact, 단순 앵커링은 direct payload

3. Proof Service 최적화
- 블록 채굴 후 비동기 워커가 증명 선계산  
- Redis로 핫캐시, 디스크 스냅샷으로 콜드 보존  
- 워커 풀로 동시성 제어하여 RocksDB 충돌 완화

4. 캐시 및 인덱스
- batchId → (blockHash, leafIndex, merklePathCached) 유지  
- 메모리 매핑 + Redis 메타로 빠른 조회 제공

5. 프로토콜 및 데이터 형식
- Batch 메타: { batchId, merkleRoot, submitter, timestamp, chainId, blockHeight, proofUri }  
- Proof 형식: VerifyProof.hs 출력 구조를 바탕으로 바이너리 및 JSON 직렬화 제공  
- API: GET /v1/proof/{batchId}, POST /v1/proof/verify

6. 보안 및 일관성
- 재생 공격 방지: timestamp + nonce 사용  
- 해시 알고리즘 버전 일치 보장(Version.hs)  
- 감사 로그: 온/오프체인에 모든 작업 기록

양자 안전성
- 현재 SHA512t_256 사용; 향후 Keccak-256 전환 고려(참조: src/Chainweb/SPV/PayloadProof.hs 의 SpvKeccak_256).  
- 해시 알고리즘 버전은 Version.hs에 기록; Chainweb.RestAPI.Utils에 nonce 추가.  
- Pact 계약에 서명 검증 로직 추가 검토.

## 7 성능 및 운영 고려사항
- 주요 병목: RocksDB I/O(컴팩션), Merkle 증명 생성의 CPU 부하.  
- 권장 지표: end-to-end p50/p95/p99, Proof QPS, 증명 생성 시간, RocksDB IOPS.  
- 배포: Edge Gateway는 엣지에, Proof Service는 Chainweb 노드와 colocate 또는 전용 인스턴스. Redis/캐시 분산 구성.

테스트 및 벤치마크
- 기존 테스트 재사용: `cabal test` (SPV), Pact SPV 통합 테스트.  
- Edge Gateway용 E2E 테스트 추가.  
- 벤치: pact-repl 및 test/lib로 Pact 실행 지연 및 증명 생성 측정.

성능 모델 요약
- Throughput 근사: T ≈ (n · t_c) / (1 + B + C), C는 캐시 히트율 요인.  
- RocksDB 파라미터는 src/Chainweb/Payload/PayloadStore/RocksDB.hs에서 조정.

## 8 권장 코드 변경 (우선순위)
고우선순 (MVP):
- src/Chainweb/SPV/CreateProof.hs: 비동기 precompute API 및 Redis 캐시 인터페이스 추가  
- src/Chainweb/Pact/RestAPI/Server.hs: 배치 쓰기 및 batchId 조회 엔드포인트 추가(또는 Edge Gateway 서비스 신규)  
- src/Chainweb/Payload/PayloadStore/RocksDB.hs: 배치 쓰기 경로 및 쓰기 튜닝 파라미터 추가

중기:
- Proof Service의 Redis 캐시 + LRU 전략 구현 (bench/의 PactService 템플릿 활용)  
- JS/Python SDK 제공

장기:
- 해시 하드웨어 가속, ZK 보조 증명 압축 연구

## 9 로드맵 (권장)
- Month 0–1: Edge Gateway API 정의, PoC(디바이스→Gateway→Chain)  
- Month 1–3: Batch SPV 플로우 및 Proof Service 기본 캐시 (MVP)  
- Month 3–6: 최적화(RocksDB/워커 풀), SDK 제공, PoC 고객 확보  
- Month 6–12: 엔터프라이즈 기능(RBAC, 모니터링), 하드웨어 가속 연구

## 10 리스크·비용 및 완화
- 리스크: I/O 병목, SPV 지연, 복잡 계약으로 인한 지연 확대  
- 완화: 배치화, 선계산, 캐시, payload vs pact 분리 아키텍처

## 11 결론
권장 접근: "Edge Gateway + Batch SPV + Pact 정산". 약 3개월 내 실용적 MVP 가능. 이후 캐시·선계산으로 프로덕션 스케일에 대응.

## 12 참고 코드 위치 (엔지니어링 참조)
- 증명 생성: src/Chainweb/SPV/CreateProof.hs  
- 증명 검증: src/Chainweb/SPV/VerifyProof.hs  
- Pact REST Server: src/Chainweb/Pact*/RestAPI/Server.hs  
- Payload 저장(RocksDB): src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- PactService 예제(bench): bench/Chainweb/Pact/Backend/PactService.hs  
- MerkleLog 예제: docs/merklelog-example.md  
- 버전 관리: src/Chainweb/Version.hs  
- 종합 보고서(중문): [`ChainwebPactSPV_종합전략평가_권고_ko.md`](./ChainwebPactSPV_종합전략평가_권고_ko.md)  
- 에코시스템: chainweb-mining-client-0.7 (DePIN 채굴 도구 후보)
