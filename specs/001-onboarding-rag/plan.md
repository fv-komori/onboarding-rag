# 実装計画: 社内ドキュメントQ&Aシステム

**ブランチ**: `001-onboarding-rag` | **日付**: 2025-12-01 | **仕様**: [spec.md](./spec.md)
**入力**: `/specs/001-onboarding-rag/spec.md`の機能仕様

**注**: このテンプレートは`/speckit.plan`コマンドで埋められる。実行フローは`.specify/templates/commands/plan.md`を参照。

## 概要

新入社員や案件異動者向けのRAGベース社内ドキュメントQ&Aシステムを構築する。ユーザーは自然言語で質問を入力し、就業規則、社内ポータル、案件固有のマニュアルから関連情報と出典を取得できる。Amazon Bedrockを活用したサーバーレスアーキテクチャにより、低運用コストで高精度な回答を実現する。

**主要要件**:
- SSOによるユーザー認証と案件ベースのアクセス制御
- PDF/Word/テキストドキュメントからのRAG検索
- 回答に情報源(ドキュメント名、ページ番号)を明示
- フィードバック機能による継続的改善
- 3秒以内の応答時間(成功基準)

## 技術スタック

### コアテクノロジー

**言語/バージョン**: Python 3.12 (AWS Lambda正式サポート、Amazon Linux 2023ベース)
**主な依存ライブラリ**:
- Backend: boto3 (AWS SDK), langchain, pydantic
- UI: Streamlit (初期フェーズ) / React (本格UI時)
- IaC: AWS CDK (Python) または AWS SAM
**データ保存**:
- ドキュメント: Amazon S3
- ベクトルDB: Amazon Bedrock Knowledge Base (OpenSearch Serverless)
- メタデータ/フィードバック: Amazon DynamoDB
**テストツール**: pytest, moto (AWS mocking), locust (負荷テスト)
**実行環境**: AWS (Lambda, API Gateway, Bedrock)
**プロジェクト種別**: Web (API + UI)
**性能目標**:
- API応答時間 < 3秒 (p95) - 仕様書SC-001より
- 100同時ユーザーで応答時間維持 - 仕様書SC-005より
- LLM応答生成時間 < 10秒 (p95)
**制約事項**:
- SSOシステムとの統合必須
- AWS Lambda実行時間制限 (15分)
- Bedrock APIレート制限考慮
**規模感**:
- 初期想定: 100-200ユーザー
- ドキュメント: 数百〜数千ファイル
- 月間クエリ数: 数千〜1万程度

### AWSアーキテクチャ詳細

**LLM**: Amazon Bedrock (Claude Sonnet 4.5推奨、2025年9月リリース)
**埋め込みモデル**: Amazon Bedrock Embeddings (Titan Embeddings v2 or Cohere Embed)
**ベクトル検索**: Amazon Bedrock Knowledge Base
  - バックエンド: OpenSearch Serverless (マネージド)
  - S3にドキュメント配置で自動同期
**API構成**:
  - Amazon API Gateway (REST API)
  - AWS Lambda (Python 3.12 runtime)
  - Lambda → Bedrock Runtime API + Knowledge Base API
**UI選択肢**:
  1. **Streamlit** (Phase 1推奨): Python単独でUI構築、迅速なプロトタイピング
     - デプロイ: ECS Fargate or App Runner
  2. **Slack Bot** (Phase 2): 既存ツール統合
     - Lambda + API Gateway + Slack Events API
  3. **React** (Phase 3): 本格的なUI
     - S3 + CloudFront (静的ホスティング)
**データ取り込みフロー**:
  - S3バケット構造:
    - `/common/` - 全社共通ドキュメント
    - `/projects/{project-id}/` - 案件別ドキュメント
  - トリガー: S3 Event → EventBridge → Lambda → Knowledge Base Sync
  - メタデータ付与: Lambda関数でS3パスから自動抽出
    - カテゴリ、部署、案件ID、アップロード日時
**フィードバック/ログ**:
  - DynamoDB: 質問履歴、評価(👍👎)、コメント
  - CloudWatch Logs: 構造化ログ (JSON)、X-Ray統合
  - CloudWatch Dashboard or Amazon QuickSight: 満足度分析
**認証**:
  - Amazon Cognito (SSOプロバイダー統合: SAML/OIDC)
  - API Gateway Cognito Authorizer

## 憲法チェック

*ゲート条件: Phase 0調査の前に通過必須。Phase 1設計後に再チェック。*

### 原則0: ドキュメント言語
**ステータス**: ✅ 準拠
- すべての仕様書、計画書、タスクは日本語で記述
- API仕様のエンドポイント名、パラメータ名は英語
- 技術用語は原語のまま使用(Lambda, Bedrock等)

### 原則I: モジュール型アーキテクチャ
**ステータス**: ✅ 準拠
提案アーキテクチャは以下の分離されたコンポーネントで構成:
- **ドキュメント取り込み層**: S3 Event → EventBridge → Lambda → Knowledge Base
- **ベクトル検索層**: Bedrock Knowledge Base (OpenSearch Serverless)
- **生成層**: Bedrock Runtime API (Claude Sonnet 4.5)
- **API層**: API Gateway + Lambda

各コンポーネントは独立してデプロイ、テスト、スケール可能。

### 原則II: サーバーレスファースト
**ステータス**: ✅ 準拠
すべてのコンポーネントがサーバーレス:
- Lambda関数: ビジネスロジック
- API Gateway: HTTPルーティング、認証
- S3: ドキュメントストレージ
- DynamoDB: メタデータ、フィードバック
- Bedrock: LLM推論
- OpenSearch Serverless: ベクトル検索(Knowledge Base経由)

**注**: Streamlit UIのみECS Fargate/App Runnerを使用(Phase 1)。ReactベースUI(Phase 3)は完全にサーバーレス(S3 + CloudFront)。

### 原則III: コスト効率とパフォーマンス
**ステータス**: ✅ 準拠
- Lambda最適化: メモリ配分調整、ProvisionedConcurrency検討
- ベクトル検索効率: Bedrock Knowledge Baseのマネージドインデックス活用
- Bedrock呼び出し最適化: プロンプトキャッシング、適切なモデル選択
- 性能目標: API応答 < 3秒(仕様書要件)、LLM応答 < 10秒

**Phase 0で要調査**:
- Bedrock Knowledge Baseの検索レイテンシ実測
- Lambda関数の最適メモリ設定
- プロンプトキャッシング戦略

### 原則IV: 観測可能性とデバッグ性
**ステータス**: ✅ 準拠
- CloudWatch Logs: 全Lambda関数から構造化ログ(JSON)
- X-Ray トレーシング: リクエストフロー全体の追跡
- メトリクス: レイテンシ、エラー率、トークン使用量、コスト
- アラート: エラー閾値、レイテンシスパイク設定

ログ内容:
- リクエストID (追跡用)
- ユーザーコンテキスト (匿名化)
- プロンプト/レスポンスメタデータ
- エラーとスタックトレース
- パフォーマンスメトリクス

### 原則V: テストとバリデーション
**ステータス**: ⚠️ Phase 1で詳細化必要
計画段階のテスト戦略:
- **ユニットテスト**: Lambda関数、メタデータ抽出ロジック (pytest)
- **統合テスト**: Lambda → Bedrock、Lambda → DynamoDB (moto)
- **コントラクトテスト**: API Gateway エンドポイント (OpenAPI検証)
- **E2Eテスト**: ドキュメントアップロード → クエリ → 回答フロー
- **負荷テスト**: 100同時ユーザーシナリオ (locust)

**Phase 1で要定義**:
- テストデータセット (サンプルPDF、Word、テキスト)
- 予想クエリと回答ペアのゴールデンデータセット
- エッジケーステストケース

### アーキテクチャ制約チェック

**クラウドプラットフォーム**: ✅ AWSサーバーレス使用、IaC (AWS CDK/SAM)予定

**データストレージ**: ✅ 準拠
- ドキュメントストレージ: S3
- ベクトル検索: Bedrock Knowledge Base (OpenSearch Serverless)
- メタデータ: DynamoDB

**API設計**: ⚠️ Phase 1で詳細定義
RESTful API構成予定:
```
POST /documents          # ドキュメントアップロード
POST /query              # RAGクエリ実行
POST /feedback           # フィードバック送信
GET  /query-history      # 質問履歴取得
```
Phase 1でOpenAPI仕様を作成。

**データフローパターン**: ✅ 準拠
- 取り込みフロー: S3アップロード → EventBridge → Lambda → Knowledge Base同期
- クエリフロー: API Gateway → Lambda → 埋め込み生成 → Knowledge Base検索 → Bedrock (Claude) → レスポンス

**セキュリティとコンプライアンス**: ✅ 準拠
- 認証: Cognito (SSO統合)
- 認可: API Gateway Cognito Authorizer、Lambda IAMロール(最小権限)
- 転送中暗号化: HTTPS、TLS
- 保管中暗号化: S3 (SSE-S3)、DynamoDB暗号化
- PII除去: CloudWatch Logsから個人情報マスキング

### ゲート判定: ✅ Phase 0に進行可能

すべての必須原則に準拠または Phase 1で詳細化予定。違反なし。

## プロジェクト構成

### ドキュメント (この機能)

```text
specs/[###-feature]/
├── plan.md              # このファイル (/speckit.plan の出力)
├── research.md          # Phase 0 の出力 (/speckit.plan が生成)
├── data-model.md        # Phase 1 の出力 (/speckit.plan が生成)
├── quickstart.md        # Phase 1 の出力 (/speckit.plan が生成)
├── contracts/           # Phase 1 の出力 (/speckit.plan が生成)
└── tasks.md             # Phase 2 の出力 (/speckit.tasks で生成 - /speckit.planでは作らない)
```

### ソースコード (リポジトリルート)

```text
# Webアプリケーション構成 (Backend + Frontend)

backend/
├── src/
│   ├── lambdas/                    # Lambda関数エントリーポイント
│   │   ├── query_handler.py        # RAGクエリ処理
│   │   ├── document_processor.py   # ドキュメント取り込み
│   │   ├── feedback_handler.py     # フィードバック記録
│   │   └── sync_kb.py              # Knowledge Base同期
│   ├── services/                   # ビジネスロジック
│   │   ├── bedrock_service.py      # Bedrock API呼び出し
│   │   ├── kb_service.py           # Knowledge Base検索
│   │   ├── metadata_service.py     # メタデータ抽出/管理
│   │   └── feedback_service.py     # フィードバック処理
│   ├── models/                     # データモデル (Pydantic)
│   │   ├── query.py
│   │   ├── document.py
│   │   └── feedback.py
│   └── utils/                      # 共通ユーティリティ
│       ├── logger.py               # 構造化ログ
│       └── validators.py
├── tests/
│   ├── unit/                       # ユニットテスト
│   ├── integration/                # 統合テスト (moto使用)
│   ├── contract/                   # APIコントラクトテスト
│   └── e2e/                        # E2Eテスト
├── infrastructure/                 # IaC (AWS CDK or SAM)
│   ├── stacks/
│   │   ├── api_stack.py
│   │   ├── storage_stack.py
│   │   └── bedrock_stack.py
│   └── app.py
└── requirements.txt

frontend/
├── streamlit/                      # Phase 1: Streamlit UI
│   ├── app.py
│   ├── components/
│   └── requirements.txt
├── react/                          # Phase 3: React UI (将来)
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── services/
│   └── package.json
└── slack-bot/                      # Phase 2: Slack Bot (将来)
    └── handler.py

docs/                               # プロジェクトドキュメント
├── architecture.md
└── deployment.md
```

**構成の決定**: Webアプリケーションアーキテクチャでbackendとfrontendを分離。backendはLambda関数とサービス層を明確に分離し、infrastructureディレクトリでIaCを管理。frontendは段階的にStreamlit → Slack Bot → Reactの順で実装予定。

## 複雑度の管理

> **憲法チェックで違反があり正当化が必要な場合のみ記入**

**ステータス**: 違反なし。すべての憲法原則に準拠。

---

## Phase 1完了後の憲法チェック再評価

### 原則V: テストとバリデーション (再評価)
**ステータス**: ✅ 準拠

Phase 1で詳細化されたテスト戦略:
- **ユニットテスト**: `tests/unit/` で各サービス、モデル検証実装予定
- **統合テスト**: `tests/integration/` でmotoを使用したAWSサービスモック
- **コントラクトテスト**: `tests/contract/` でOpenAPI仕様検証 (api-spec.yaml)
- **E2Eテスト**: `tests/e2e/` で完全なユーザージャーニー検証
- **負荷テスト**: locustで100同時ユーザーシナリオ実行予定

**テストデータセット** (quickstart.mdで定義):
- サンプルドキュメント: PDF, Word, テキスト各種
- ゴールデンクエリセット: よくある質問とその期待回答
- エッジケース: 大容量ファイル、特殊文字、マルチバイト文字

### API設計 (再評価)
**ステータス**: ✅ 準拠

contracts/api-spec.yamlでOpenAPI 3.0仕様を定義完了:
- RESTful API設計
- 標準HTTPステータスコード使用
- JSON構造化データ
- ページネーションサポート
- Cognito Authorizer認証

### データフローパターン (再評価)
**ステータス**: ✅ 準拠

data-model.mdでエンティティ関係と状態遷移を明確化:
- 取り込みフロー: S3 → EventBridge → Lambda → Knowledge Base
- クエリフロー: API Gateway → Lambda → Bedrock (Embedding + LLM) → レスポンス
- すべてのエンティティ(Document, Query, Answer, Feedback)に状態遷移定義

### 最終ゲート判定: ✅ Phase 2 (タスク生成) に進行可能

すべての憲法原則に準拠。Phase 1設計完了。
