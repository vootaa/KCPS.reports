# Chainweb + Pact + SPV 導入シナリオ分析とシステム設計（日本語）

バージョン：コードベース chainweb-node-2.31.1 に基づく  
作成者：GitHub Copilot（戦略技術コンサルタント風）  
日付：2025-10-30

## 1 要約（意思決定者向けサマリ）
Chainweb は並列かつ高い検証性を持つマルチチェーン基盤を提供します。ネイティブな SPV サポートと Pact コントラクトシステムにより、「検証可能なイベントアンカリング」「監査可能な決済」「ライトクライアント検証」用途に適しています。IoT、DePIN、AI の検証可能な計算/データ市場向けに、MVP としては「Edge Gateway + バッチ SPV API + Pact による決済」を推奨します。まずは SPV 証明生成の遅延と I/O ボトルネックを解決することを優先します。

## 2 既存の能力（ソースコードに基づく）
- 並列マルチチェーンコア：src/Chainweb/*（chain、graph、バージョン管理）  
- SPV 証明生成・検証：src/Chainweb/SPV/CreateProof.hs、src/Chainweb/SPV/VerifyProof.hs  
- Pact 統合および REST インタフェース：src/Chainweb/Pact*/RestAPI/Server.hs、Bench の PactService サンプル  
- 永続化層と I/O：src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- 検証可能ログの例：docs/merklelog-example.md

これらのモジュールは「要約を送信 → バッチでチェーンに格納 → SPV 証明生成 → 外部で検証」というフローを直接サポートします。

### Pact バージョン差異と互換性
- Chainweb は Pact4/Pact5 の切替をサポートしています（chainweb-node-2.31.1/src/Chainweb/Version.hs の PactVersion 型参照）。  
- Pact5（pact-5-5.4）は Gas 計算や SPV 検証を最適化しており（verify-spv 関数参照）、高スループットシナリオに適します。コントラクトはデプロイ時にバージョンを明示する必要があります。  
- 新規開発には Pact5 を推奨します。既存の Pact5 用 SPV テストも再利用可能です。  
- バージョン間の呼び出しは Chainweb.Pact.Conversion で取り扱う必要があります。

## 3 適用シナリオと提供価値
- IoT / DePIN：多数のデバイスからのイベントを Merkle root としてチェーンに登録し、SPV 証明で監査や契約決済を行う。信頼コスト低減と不変な証拠を提供。  
- AI（検証可能な計算）とデータ市場：タスクのメタデータや結果の要約をアンカーして Pact で自動決済。透明な価格付けと責任の所在を提供。  
- SPV-as-a-Service：証明生成・検証 API を提供し、ライトクライアントのニーズを軽減。収益化しやすい。  
- ゲーム / GameFi：アイテムや試合結果のアンカリング、仲裁用の可証拠。  
- 企業向け監査サービス：不変の証跡、コンプライアンス報告の自動化。

## 3.1 ゲーム向けシナリオ（Game / GameFi）

ユースケース概要
- アイテムや資産のチェーン上登録（NFT 等）、高並列処理に対応。  
- リアルタイム近傍のイベントをバッチでアンカーして外部マーケットや仲裁に証拠を提供。  
- サーバ間/ワールド間の資産移転はチェーン分割（chainId による）と SPV/フォワーダを用いる。  
- トーナメント決済は Pact コントラクトで自動化、MerkleRoot をオンチェーンに残し Proof Service が裁定用証拠を供給。

高レベルアーキテクチャ（概要）
- ゲームサーバ（低遅延ロジック）→ Edge Gateway（集約、Merkle 枝作成、chainId ルーティング）→ Anchor/Batch Ingestor（Pact や payload を書き込む）→ Proof Service（ブロック確定後に証明を事前計算・キャッシュ）→ Asset Vault（Pact 契約で管理）→ Marketplace/Wallet（Proof を要求）。

設計上のトレードオフ
- レイテンシ vs 最終性：リアルタイム状態はオフチェーンで扱い、重要な経済イベントはオンチェーンで確定。  
- マルチチェーン分割：世界/リージョンごとに chain を割当て、クロスチェーンには two-phase パターンを採用。  
- キャッシュとロールバック：オフチェーンとチェーンの不整合時は合約仲裁や補正トランザクションで解決。  
- セキュリティ：Edge Gateway / Proof Service で署名検証やレート制御を実装。

簡易フロー
1. プレイヤーがスキンを購入 → GameServer が Edge Gateway にイベント送信  
2. Edge Gateway がバッチして Pact の recordBatch 等に MerkleRoot を書き込む  
3. 確認後、Proof Service が証明を事前計算し Marketplace が要求して移転/表示を完了

パフォーマンス考察
- 書込み競合はマルチチェーン分割で対処。証明生成がボトルネックなら遅延確認とキャッシュを利用。  
- 高負荷時は複数 Proof Service を Redis シャーディングで分散。

## 3.2 監査（Enterprise Audit / Compliance）

ユースケース
- コンプライアンス証跡や請求・ログのアンカリング、不可変で検証可能な時系列証拠を提供。  
- サプライチェーンの各段階を MerkleRoot として記録し、第三者が SPV で検証。  
- Pact によるルールで自動通知やトリガーを行い、証明を監査証憑として保存。

高レベル構成
- Data Ingestors（ETL、署名付きイベント）→ Normalizer & Policy Engine（書式統一と入チェーンポリシー）→ Audit Anchor Service（バッチ化してチェーンへ）→ Proof Catalog & Index（batchId→ブロック情報・証明 URI を保存）→ Audit Portal（検証 UI/CLI）→ Archival Layer（暗号化して cold storage に保管）。

マルチチェーンとテナント隔離
- 顧客ごとに chainId を割当てるか、replicated コントラクトに tenantId を持たせる。監督用途では replicated を推奨。

セキュリティとコンプライアンス
- Chain of custody：イベントは ingest 時に署名し submitter の ID を付与。Pact と Portal で RBAC を実装。敏感データは暗号化してオンチェーンには最小メタデータのみ保持。

API 例
- POST /v1/audit/batch { events[], policy, tenantId } -> { batchId, submittedToChain, txHash }  
- GET /v1/audit/proof/{batchId} -> { proof, blockHash, timestamp, verifierHints }  
- GET /v1/audit/catalog?... -> バッチの一覧

SLA と運用指標
- 例：proof 可用性 99.9%（TTL 24h）、長期保存 7–10 年。監視項目は未処理数、証明生成遅延、cold storage 取り出し時間等。

監査フロー例
1. 企業システムが署名付きログを Ingestor に送信  
2. Anchor Service がバッチで入チェーンし batchId と txHash を返却  
3. 監査員が Portal で proof を要求し Proof Service が提供。クライアント側で VerifyProof を実行。

拡張提案
- TSA（タイムスタンプ認証局）と連携し時刻の強化を行う。  
- 長期保管のため定期的に再アンカリングすることでハッシュアルゴリズム寿命リスクを軽減。

## 3.3 IoT / DePIN

ユースケース
- 大規模テレメトリやメータリング：スマートメーターや環境センサーのイベントをバッチで上鏈。  
- デバイス認証と利用権証明：デバイス登録や証明書の発行・使用履歴の検証。  
- デバイス間のマイクロペイメント：Pact で決済し SPV で支払証拠を提供。

専用アーキテクチャ
- Device SDK（軽量）：署名、パッケージング、リトライ、オプションでローカル Merkle 構築。  
- Edge Gateway：TLS/mTLS、フロー制御、重複排除、キャッシュ、deterministic routing による chainId 振り分け。バッチは時間窓または容量で設定。  
- 書込み戦略：通常イベントは direct payload（低遅延）、重要イベントは Pact（契約決済）。  
- Proof & Verification：Proof Service は batchId に基づく証明取得と検証を提供、JS/Python SDK に検証機能を組込む。

実装要点
- 地理的にエッジと Proof を近接配置することを推奨。  
- 元データは暗号化して cold storage に保管、チェーンには MerkleRoot のみを残す。  
- at-least-once を保証し idempotencyKey で重複を防ぐ。

セキュリティとプライバシー
- デバイス ID ライフサイクル管理（失効/更新）。  
- オンチェーンには最小限の要約のみ、敏感情報はオフチェーンで管理。

指標とチューニング
- 推奨バッチウィンドウ 0.5–2s、サイズ 500–2000 イベント、目標 p95 ≤ 2s（MVP）。

## 3.4 AI：検証可能な計算とデータ市場

ユースケース
- タスク登録と結果のアンカリング（トレーニング、推論結果、メタデータ）。  
- モデル市場：仕事のスコープ定義、提供者が実行証明を提出して Pact で支払い。  
- データセットの出所検証：提出サンプルのタイムスタンプと原本性の証明。

アーキテクチャ（概要）
- Task Broker：タスクの登録・マッチング・配布。  
- Compute Node：実作業を実行し結果要約を署名して返却。  
- Result Anchoring：Edge Gateway が Merkle バッチを生成して Chainweb に書込む。高価値タスクは Pact で保証。  
- Incentive & Settlement：Pact が仲裁・支払条件を保持。Proof Service が証明のチェーンを提供。

実装ポイント
- 大きな出力は Compute Node 側でスライスの Merkle を作り root をアンカリング。  
- 実行環境ログも Merkle に含め不正対策のための監査を可能にする。  
- レイテンシ別のパス（fast-path / final-path）を用意。

データガバナンス
- サンプルは暗号化してから送信、チェーンには最小情報のみ。Proof Catalog で追跡可能にする。

商業モデル
- proof リクエスト課金、決済手数料、証明保管サービス等で収益化。

## 3.5 SPV-as-a-Service

ユースケース
- ウォレット、ブリッジ、監査向けにオンデマンドで SPV 証明生成・検証 API を提供。  
- SLA、保管、RBAC、準拠証明を含むホワイトラベルサービスの提供。

アーキテクチャ
- Public API Gateway：REST/gRPC、認証、課金。  
- Proof Fabric：ワーカープール、優先度キュー、Redis + オブジェクトストレージでキャッシュ・保持。  
- マルチテナント：ネームスペース、アクセス制御、監査ログ。  
- 可用性：ホットスタンバイ、リージョン間レプリケーション。

主要設計点
- on-demand vs precompute のトレードオフを明確化。  
- 悪用対策：レート制限、サイズ/複雑度に基づく課金。  
- 透明性：生成ログやビルドハッシュ等を提供し信頼性を高める。

コンプライアンス
- 証明の出自メタデータ（生成ノード、バージョン、タイムスタンプ）を保存。必要に応じてチェーン上にサービスログを残す。

運用と収益化
- 価格モデル：リクエスト毎、サイズ別、サブスクリプション等。企業向けは SLA と長期アーカイブを追加。

kda-tool 統合
- 企業向けに kda-tool gen でマルチチェーン取引を生成し Chainweb の SPV ワークフローに統合。kda-tool の Docker イメージを chainweb-node と同一環境で運用し CLI による署名と検証を提供。

## 4 技術ソリューション（概要）
目標：最終的一貫性を担保しつつスループットを最大化し、検証遅延を最小化する。戦略：「エッジで集約 → バッチでチェーンに書込 → 証明を事前計算・キャッシュ」。

主要コンポーネント：
- Edge Gateway：イベント受信、バッチ化、署名検証、レート制御。  
- Batch SPV Ingestor：MerkleRoot を Pact 或いは直接ペイロードで Chainweb に書込む。  
- SPV Proof Service：CreateProof モジュールを用い証明を生成しキャッシュする。  
- Pact Contract Layer：登録や決済用のコントラクトテンプレート。  
- Light Client SDK：証明検証ライブラリ（JS/Python/Go）。  
- Observability：TPS/遅延/I/O メトリクス。

## 5 システム高位アーキテクチャ（コンポーネントとデータフロー）
コンポーネント一覧：
1. デバイス / エッジデバイス：イベント→署名→Edge Gateway  
2. Edge Gateway：キュー、デデュープ、バッチ、MerkleRoot 作成→Batch SPV Ingestor に送信  
3. Chainweb ノード群：バッチを受信しブロックを生成  
4. SPV Proof Service：CreateProof を実行しキャッシュを提供、REST API で getProof(batchId) 等を公開  
5. Pact コントラクト：メタデータや決済を管理  
6. ライトクライアント：proof を取得してローカルで検証

データフロー：デバイス → Edge Gateway（バッチ）→ Chainweb 書込 → ブロック確定 → Proof Service が証明を生成/キャッシュ → クライアントが取得し検証

## 5.1 マルチチェーン戦略と抽象化レイヤ

問い：各チェーンに同一コントラクトを展開すべきか？タスクをどう分散するか？上位層に透過化できるか？

結論（要点）
- コントラクトは replicated（全チェーン同一）または partitioned（分割）でデプロイ可能。replicated は簡素だが全体一致性が必要な場合は追加同期が必要。partitioned はスケールに有利だがクロスチェーン操作が複雑。  
- ルーティングは deterministic hash（例：H(taskKey) mod N）か、一致ハッシュ（consistent hashing）を推奨。  
- Virtual Chain Layer（VCL）を提案：上位に透過的な Submit/Query/Call API を提供し内部でルーティング・再試行・証明検証を行う。

詳細戦略（概略）
1) デプロイモード：replicated vs partitioned のメリット/デメリットを整理  
2) ルーティング：chainId = H(taskKey) % N を基本に、拡張で sticky / load-aware を実装  
3) VCL 設計：Router、Executor、Verifier/Coordinator の三層で実装し submitTask/getResult/callCrossChain 等の API を提供  
4) 呼び出しパターン：オンチェーンフォワーダ（検証と実行をチェーン内で完結）か、オフチェーンルータ + 軽量オンチェーン検証の選択  
5) クロスチェーン整合性：弱整合（デフォルト）と強整合（2PC や proof ベースの commit）をケース別に採用

実装候補モジュール位置
- 新規モジュール例：src/Chainweb/Multichain/Router.hs  
- 既存の CreateProof/VerifyProof、Pact REST Server、PayloadStore/RocksDB を再利用

MVP パス（優先順）
1. deterministic routing（hash mod N）を実装し Edge Gateway が chainId を返す  
2. Edge Gateway/Router にシングルチェーン submit をラップする機能を実装  
3. オフチェーン aggregator によるクロスチェーン読み取りと proof relay を実装

リスクとトレードオフ
- クロスチェーンは遅延とコストを増す。低レイテンシ用途は partitioned を優先。  
- replicated はローンチ容易だが状態分散の reconciliate が必要。  
- オフチェーン Router を信頼する場合は多重化/監査ログで信頼境界を下げる設計が必要。

擬似コード（要旨）
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

クロスチェーン（proof relay）
```pseudo
// callCrossChain(fromChain, toChain, txHash, callPayload)
proof = CreateProof(fromChain, txHash)
verifyLocally = VerifyProof(proof)
if verifyLocally:
  submitToChainWithProof(toChain, callPayload, proof)
else:
  fail("proof invalid")
```

優先実装項目
1. deterministic routing と Edge Gateway の chainId 応答（MVP）  
2. Proof Service の off-chain relay と VCL API の公開  
3. 必要な場合に two-phase pattern の実装

## 6 モジュール別の詳細実装提案
1. Edge Gateway
   - API：POST /v1/events/batch { events[], batchSize, ttl } → batchId  
   - バッチ戦略：時間窓（例 1s）または容量（例 1000 events）  
   - Merkle はチェーンと同一ハッシュ関数を使用（Version.hs を確認）  
   - 署名検証・submitter metadata の記録

2. バッチのチェーン書込
   - パス a) Pact 呼出（契約で batchId を管理）  
   - パス b) 直接 payload 書込（低遅延だが決済連携には別途メタデータ登録が必要）  
   - 推奨：決済が必要な場合は Pact、単なるアンカリングは payload を利用

3. Proof Service 最適化
   - ブロック生成後に非同期 worker が証明を事前計算  
   - Redis をホットキャッシュ、ディスクにスナップショットを保管  
   - ワーカープールで生成並列度を制御し RocksDB 競合を防ぐ

4. キャッシュとインデックス
   - batchId → (blockHash, leafIndex, merklePathCached) を保持  
   - メモリマップ + Redis メタデータで高速検索

5. プロトコルとデータ形式
   - Batch メタ：{ batchId, merkleRoot, submitter, timestamp, chainId, blockHeight, proofUri }  
   - Proof 形式：VerifyProof.hs 出力に基づくバイナリと JSON シリアライズの両対応  
   - API 例：GET /v1/proof/{batchId}, POST /v1/proof/verify

6. セキュリティと一貫性
   - リプレイ防止：timestamp + nonce を導入  
   - ハッシュアルゴリズムのバージョン一致を保証（Version.hs）  
   - 監査ログをチェーン上/オフチェーンで保存

量子耐性の考慮
- 現行は SHA512t_256 を利用しているが、将来的に Keccak-256 への移行を検討（src/Chainweb/SPV/PayloadProof.hs の SpvKeccak_256 オプション参照）。  
- ハッシュアルゴリズムのバージョン管理を Version.hs に記録し、Chainweb.RestAPI.Utils に nonce を実装。

## 7 性能と運用上の考察
- 主なボトルネック：RocksDB の I/O（コンパクション）、Merkle 証明生成の CPU 負荷。  
- 推奨メトリクス：end-to-end p50/p95/p99、Proof QPS、証明生成時間、RocksDB IOPS。  
- デプロイ戦略：Edge Gateway をエッジに、Proof Service を Chainweb ノードにコロケートまたは専用配置。Redis/Cache は分散化。

テストとベンチマーク
- 既存テストを再利用：cabal test（SPV 関連）や Pact SPV 統合テスト。  
- 新規 Edge Gateway 用の E2E テストを追加。  
- ベンチは pact-repl と test/lib を活用し Pact 実行遅延や証明生成を測定。

基本的な性能モデル（概略）
- スループット近似：T ≈ (n · t_c) / (1 + B + C)、C はキャッシュヒット要因  
- RocksDB 設定で IOPS を監視し調整

## 8 推奨コード変更（優先度付け）
高優先度（MVP）：
- src/Chainweb/SPV/CreateProof.hs に非同期 precompute API とキャッシュ（Redis）インタフェースを追加  
- src/Chainweb/Pact/RestAPI/Server.hs にバッチ書込と batchId 検索エンドポイントを追加（または Edge Gateway サービスを新設）  
- src/Chainweb/Payload/PayloadStore/RocksDB.hs にバッチ書込パスと書込チューニング設定を追加

中期：
- Proof Service の Redis キャッシュ層と LRU 戦略（bench の PactService を参考に実装）  
- SDK（JS/Python）の提供

長期：
- ハードウェアアクセラレーション、ZK 補助による証明圧縮の研究

## 9 ロードマップ（例）
- Month 0–1：Edge Gateway API 定義、PoC（デバイス→Gateway→Chain）  
- Month 1–3：Batch SPV フローと Proof Service（MVP）  
- Month 3–6：チューニング（RocksDB/ワーカープール）、SDK、PoC クライアント  
- Month 6–12：企業機能（RBAC、監視）、ハードウェア加速検討

## 10 リスクと緩和策
- リスク：I/O ボトルネック、SPV の遅延、複雑コントラクトによる遅延増大  
- 緩和：バッチ化、事前計算、キャッシュ、payload と pact のレイヤ分離

## 11 結論
「Edge Gateway + Batch SPV + Pact による決済」を中心路線とし、約 3 ヶ月で実用的な MVP を提供する。以降、キャッシュと事前計算で本番スケールに対応する。

## 12 参考コード位置
- 証明生成：src/Chainweb/SPV/CreateProof.hs  
- 証明検証：src/Chainweb/SPV/VerifyProof.hs  
- Pact REST Server：src/Chainweb/Pact*/RestAPI/Server.hs  
- Payload ストア（RocksDB）：src/Chainweb/Payload/PayloadStore/RocksDB.hs  
- PactService サンプル（bench）：bench/Chainweb/Pact/Backend/PactService.hs  
- MerkleLog 例：docs/merklelog-example.md  
- バージョン管理：src/Chainweb/Version.hs  
- 総合レポート：[`ChainwebPactSPV総合戦略評価と実装提言レポート_jp.md`](./ChainwebPactSPV総合戦略評価と実装提言レポート_jp.md)。   
- エコシステム：chainweb-mining-client-0.7（DePIN 用の採掘ツール候補）
