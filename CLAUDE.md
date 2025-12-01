# 社内ドキュメントQ&Aシステム 開発ガイド

全機能計画から自動生成。最終更新: 2025-12-01

## 使用技術

### バックエンド
- **言語**: Python 3.12 (AWS Lambda正式サポート)
- **フレームワーク**: AWS Lambda (サーバーレス)
- **主要ライブラリ**:
  - `boto3` (AWS SDK)
  - `langchain` (RAGフレームワーク)
  - `pydantic` (データバリデーション)
- **テスト**: pytest, moto (AWS mocking), locust (負荷テスト)

### AWS サービス
- **LLM**: Amazon Bedrock (Claude Sonnet 4.5 - 2025年9月リリース)
- **埋め込み**: Amazon Bedrock Embeddings (Titan Embeddings V2)
- **ベクトルDB**: Bedrock Knowledge Base (OpenSearch Serverless)
- **API**: Amazon API Gateway (REST API)
- **データストア**: Amazon DynamoDB, Amazon S3
- **認証**: Amazon Cognito (SSO統合)
- **ログ/監視**: CloudWatch Logs, X-Ray, CloudWatch Dashboards
- **IaC**: AWS CDK (Python)

### フロントエンド (段階的実装)
- **Phase 1**: Streamlit (Python、MVP)
- **Phase 2**: Slack Bot (Lambda + Slack Events API)
- **Phase 3**: React (本格UI、S3 + CloudFront)

## プロジェクト構成

```text
onboarding-rag/
├── backend/
│   ├── src/
│   │   ├── lambdas/              # Lambda関数エントリーポイント
│   │   │   ├── query_handler.py  # RAGクエリ処理
│   │   │   ├── document_processor.py  # ドキュメント取り込み
│   │   │   ├── feedback_handler.py    # フィードバック記録
│   │   │   └── sync_kb.py        # Knowledge Base同期
│   │   ├── services/             # ビジネスロジック
│   │   │   ├── bedrock_service.py     # Bedrock API呼び出し
│   │   │   ├── kb_service.py          # Knowledge Base検索
│   │   │   ├── metadata_service.py    # メタデータ抽出/管理
│   │   │   └── feedback_service.py    # フィードバック処理
│   │   ├── models/               # データモデル (Pydantic)
│   │   │   ├── query.py
│   │   │   ├── document.py
│   │   │   └── feedback.py
│   │   └── utils/                # 共通ユーティリティ
│   │       ├── logger.py         # 構造化ログ
│   │       └── validators.py
│   ├── tests/
│   │   ├── unit/                 # ユニットテスト
│   │   ├── integration/          # 統合テスト (moto)
│   │   ├── contract/             # APIコントラクトテスト
│   │   └── e2e/                  # E2Eテスト
│   ├── infrastructure/           # IaC (AWS CDK)
│   │   ├── stacks/
│   │   │   ├── api_stack.py
│   │   │   ├── storage_stack.py
│   │   │   └── bedrock_stack.py
│   │   └── app.py
│   └── requirements.txt
├── frontend/
│   ├── streamlit/                # Phase 1: Streamlit UI
│   │   ├── app.py
│   │   ├── components/
│   │   └── requirements.txt
│   ├── slack-bot/                # Phase 2: Slack Bot
│   │   └── handler.py
│   └── react/                    # Phase 3: React UI
│       └── src/
├── specs/
│   └── 001-onboarding-rag/
│       ├── spec.md               # 機能仕様
│       ├── plan.md               # 実装計画
│       ├── research.md           # 技術調査結果
│       ├── data-model.md         # データモデル
│       ├── quickstart.md         # セットアップガイド
│       └── contracts/
│           └── api-spec.yaml     # OpenAPI仕様
└── .specify/
    └── memory/
        └── constitution.md       # プロジェクト憲法
```

## コマンド

### 開発環境セットアップ
```bash
# Python仮想環境
python3.12 -m venv .venv
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate     # Windows

# 依存パッケージインストール
pip install -r backend/requirements.txt
pip install -r backend/requirements-dev.txt
```

### AWS CDK
```bash
# CDKインストール
npm install -g aws-cdk

# ブートストラップ (初回のみ)
cdk bootstrap aws://<account-id>/ap-northeast-1

# デプロイ
cd backend/infrastructure
cdk deploy OnboardingRagStorageStack-dev
cdk deploy OnboardingRagBedrockStack-dev
cdk deploy OnboardingRagApiStack-dev

# すべてデプロイ
cdk deploy --all
```

### テスト
```bash
# ユニットテスト
pytest tests/unit/ -v

# カバレッジ
pytest tests/unit/ --cov=src --cov-report=html

# 統合テスト
pytest tests/integration/ -v

# E2Eテスト
pytest tests/e2e/ -v --env dev
```

### Streamlit (ローカル開発)
```bash
cd frontend/streamlit
streamlit run app.py
# http://localhost:8501
```

### AWS CLI
```bash
# Lambda関数確認
aws lambda list-functions --query "Functions[?starts_with(FunctionName, 'onboarding-rag')]"

# CloudWatch Logsの確認
aws logs tail /aws/lambda/onboarding-rag-query-handler-dev --follow

# Bedrock Knowledge Base同期確認
aws bedrock-agent list-ingestion-jobs --knowledge-base-id ${KB_ID}
```

## コーディング規約

### Python (PEP 8準拠)
- **フォーマッター**: black (line-length=100)
- **リンター**: ruff
- **型チェック**: mypy
- **インポート順**: 標準ライブラリ → サードパーティ → ローカル

### 命名規則
- **関数/変数**: snake_case
- **クラス**: PascalCase
- **定数**: UPPER_SNAKE_CASE
- **プライベート**: _leading_underscore

### ドキュメント
- すべてのpublic関数にdocstring (Google形式)
- 複雑なロジックには日本語コメント
- Pydanticモデルにはフィールド説明必須

### ログ
- 構造化ログ (JSON形式)
- ログレベル: DEBUG, INFO, WARNING, ERROR
- 必須フィールド: request_id, user_id, timestamp

### エラーハンドリング
- カスタム例外を定義 (`exceptions.py`)
- Lambda関数でエラーをキャッチし、適切なHTTPステータス返却
- CloudWatch Logsにスタックトレース出力

## 最近の更新

### 001-onboarding-rag (2025-12-01)
- **追加内容**:
  - Amazon Bedrock (Claude Sonnet 4.5) を使用したRAGシステムの基盤設計
  - Bedrock Knowledge Base + OpenSearch Serverlessによるベクトル検索
  - DynamoDB (5テーブル) でメタデータ、フィードバック管理
  - API Gateway + Lambda によるサーバーレスAPI
  - Streamlit UIでMVP構築
  - Cognito SSO統合
- **技術スタック**: Python 3.12, AWS Bedrock, Lambda, DynamoDB, S3, API Gateway
- **仕様書**: [specs/001-onboarding-rag/spec.md](specs/001-onboarding-rag/spec.md)
- **実装計画**: [specs/001-onboarding-rag/plan.md](specs/001-onboarding-rag/plan.md)

<!-- 手動追記ここから -->
<!-- 手動追記ここまで -->
