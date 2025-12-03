# クイックスタート: 社内ドキュメントQ&Aシステム

**ブランチ**: `001-onboarding-rag` | **日付**: 2025-12-01

## 概要

このドキュメントは、開発環境のセットアップから初回デプロイまでの手順を説明する。

---

## 前提条件

### 必須ツール

- **Python**: 3.12以上 (推奨: 3.12、Lambda runtime対応)
- **AWS CLI**: 2.x以上
- **Node.js**: 18.x以上 (AWS CDK用)
- **Git**: 2.x以上

### AWSアカウント要件

- AWS アカウント (管理者権限またはそれに準ずる権限)
- 以下のサービスへのアクセス権限:
  - Amazon Bedrock (Claude 3.5 Sonnet, Titan Embeddings V2モデルアクセス有効化済み)
  - Amazon S3
  - Amazon DynamoDB
  - AWS Lambda
  - Amazon API Gateway
  - Amazon Cognito
  - Amazon CloudWatch
  - AWS IAM

### Bedrock モデルアクセス有効化

初回のみ、Bedrockコンソールでモデルアクセスを有効化する必要がある:

1. AWS Management Console → Amazon Bedrock → Model access
2. 以下のモデルのアクセスをリクエスト:
   - `Claude Sonnet 4.5` (anthropic.claude-sonnet-4-5-20250929-v1:0) - 2025年9月リリース
   - `Titan Embeddings V2` (amazon.titan-embed-text-v2:0)
3. 承認まで数分待機(通常は即座に承認)

**注**: Claude 4.5シリーズは2025年に順次リリースされた最新モデルです。Sonnet 4.5はコストと性能のバランスが優れています。

---

## ローカル開発環境のセットアップ

### 1. リポジトリのクローン

```bash
git clone <repository-url>
cd onboarding-rag
git checkout 001-onboarding-rag
```

### 2. Python仮想環境のセットアップ

```bash
# 仮想環境作成
python3.12 -m venv .venv

# 仮想環境有効化 (macOS/Linux)
source .venv/bin/activate

# 仮想環境有効化 (Windows)
.venv\Scripts\activate

# 依存パッケージインストール
pip install -r backend/requirements.txt
pip install -r backend/requirements-dev.txt
```

### 3. AWS認証情報の設定

```bash
# AWS CLIで認証情報設定
aws configure

# または環境変数で設定
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export AWS_DEFAULT_REGION=ap-northeast-1  # 東京リージョン推奨
```

### 4. AWS CDKのセットアップ

```bash
# CDK CLIのインストール (グローバル)
npm install -g aws-cdk

# CDKブートストラップ (初回のみ)
cd backend/infrastructure
cdk bootstrap aws://<account-id>/ap-northeast-1

# CDK依存パッケージインストール
pip install -r requirements.txt
```

---

## 環境変数の設定

### 開発環境用 `.env` ファイル作成

`backend/.env` ファイルを作成:

```bash
# 環境
ENV=dev

# AWS
AWS_REGION=ap-northeast-1
AWS_ACCOUNT_ID=<your-account-id>

# S3
DOCUMENTS_BUCKET_NAME=onboarding-rag-documents-dev

# Bedrock
BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-5-20250929-v1:0
BEDROCK_EMBEDDING_MODEL_ID=amazon.titan-embed-text-v2:0

# Knowledge Base (デプロイ後に設定)
KB_ID=<knowledge-base-id>

# DynamoDB テーブル名
DOCUMENTS_TABLE=Documents-dev
QUERIES_TABLE=Queries-dev
ANSWERS_TABLE=Answers-dev
FEEDBACK_TABLE=Feedback-dev
PROJECTS_TABLE=Projects-dev

# Cognito (デプロイ後に設定)
COGNITO_USER_POOL_ID=<user-pool-id>
COGNITO_CLIENT_ID=<client-id>

# ログレベル
LOG_LEVEL=INFO
```

---

## インフラストラクチャのデプロイ

### 1. CDK Stackの確認

```bash
cd backend/infrastructure

# 利用可能なスタックを確認
cdk list

# 出力例:
# OnboardingRagStorageStack-dev
# OnboardingRagBedrockStack-dev
# OnboardingRagApiStack-dev
```

### 2. 段階的デプロイ

```bash
# Step 1: ストレージスタック (S3, DynamoDB)
cdk deploy OnboardingRagStorageStack-dev

# 出力からバケット名をコピーし、.envのDOCUMENTS_BUCKET_NAMEに設定

# Step 2: Bedrockスタック (Knowledge Base)
cdk deploy OnboardingRagBedrockStack-dev

# 出力からKnowledge Base IDをコピーし、.envのKB_IDに設定

# Step 3: APIスタック (Lambda, API Gateway, Cognito)
cdk deploy OnboardingRagApiStack-dev

# 出力からCognito User Pool IDとClient IDをコピーし、.envに設定
```

### 3. デプロイ確認

```bash
# Lambda関数の確認
aws lambda list-functions --query "Functions[?starts_with(FunctionName, 'onboarding-rag')]"

# API Gateway エンドポイントの確認
aws apigateway get-rest-apis --query "items[?name=='OnboardingRagApi-dev']"

# Knowledge Base の確認
aws bedrock-agent list-knowledge-bases
```

---

## 初期データのセットアップ

### 1. プロジェクト情報の登録

DynamoDBに初期プロジェクトデータを投入:

```bash
cd backend/scripts

# プロジェクトデータ投入スクリプト実行
python seed_projects.py --env dev

# 例: project-001, project-002を登録
```

### 2. テスト用ドキュメントのアップロード

```bash
# サンプルドキュメントをS3にアップロード
aws s3 cp test-data/common/hr/就業規則.pdf \
  s3://${DOCUMENTS_BUCKET_NAME}/common/hr/就業規則.pdf

# Knowledge Base同期トリガー (自動)
# S3 Event → EventBridge → Lambda → Knowledge Base
```

### 3. Knowledge Base同期の確認

```bash
# 同期ジョブのステータス確認
aws bedrock-agent list-ingestion-jobs \
  --knowledge-base-id ${KB_ID} \
  --data-source-id ${DATA_SOURCE_ID}

# ステータスが "COMPLETE" になるまで待機 (数分)
```

---

## Streamlit UIの起動 (ローカル開発)

### 1. Streamlit依存パッケージのインストール

```bash
cd frontend/streamlit
pip install -r requirements.txt
```

### 2. Streamlit設定ファイル作成

`frontend/streamlit/.streamlit/secrets.toml` を作成:

```toml
[aws]
region = "ap-northeast-1"
api_gateway_url = "https://<api-id>.execute-api.ap-northeast-1.amazonaws.com/dev"

[cognito]
user_pool_id = "<user-pool-id>"
client_id = "<client-id>"
```

### 3. Streamlit起動

```bash
streamlit run app.py

# ブラウザで http://localhost:8501 にアクセス
```

---

## テスト実行

### ユニットテスト

```bash
cd backend

# すべてのユニットテストを実行
pytest tests/unit/ -v

# カバレッジレポート生成
pytest tests/unit/ --cov=src --cov-report=html
```

### 統合テスト (moto使用)

```bash
# DynamoDB、S3のモックを使用した統合テスト
pytest tests/integration/ -v
```

### コントラクトテスト (OpenAPI検証)

```bash
# OpenAPI仕様とLambdaレスポンスの整合性チェック
pytest tests/contract/ -v
```

### E2Eテスト

```bash
# 実際のAWS環境に対するE2Eテスト (dev環境)
pytest tests/e2e/ -v --env dev
```

---

## 使用方法

### 1. Cognitoユーザーの作成

```bash
# 管理者ユーザー作成
aws cognito-idp admin-create-user \
  --user-pool-id ${COGNITO_USER_POOL_ID} \
  --username test-user@example.com \
  --user-attributes Name=email,Value=test-user@example.com \
                     Name=name,Value="Test User" \
                     Name=custom:department,Value="engineering" \
                     Name=custom:projects,Value="project-001,project-002" \
  --temporary-password "TempPassword123!"

# 初回ログイン後、パスワード変更が必要
```

### 2. Streamlit UIからログイン

1. http://localhost:8501 にアクセス
2. Cognitoログイン画面でtest-user@example.comとパスワード入力
3. 初回ログイン時はパスワード変更

### 3. 質問の実行

1. 質問入力欄に「経費精算の方法は?」と入力
2. 案件選択ドロップダウンから対象案件を選択(任意)
3. 「質問を送信」ボタンをクリック
4. 回答と参照元ドキュメントが表示される

### 4. フィードバック送信

1. 回答の下部に表示される「役に立った」「役に立たなかった」ボタンをクリック
2. (任意) コメント欄にフィードバックを入力
3. 「送信」ボタンをクリック

---

## トラブルシューティング

### Bedrockモデルアクセスエラー

**エラー**:
```
AccessDeniedException: You don't have access to the model with the specified model ID.
```

**解決方法**:
1. Bedrockコンソールで該当モデルのアクセスが有効化されているか確認
2. IAMロールに `bedrock:InvokeModel` 権限があるか確認

### Knowledge Base同期が失敗する

**エラー**:
```
Ingestion job failed with status: FAILED
```

**解決方法**:
1. CloudWatch Logsで詳細エラーを確認
2. S3オブジェクトのファイル形式が対応しているか確認(PDF, DOCX, TXT, MD)
3. S3バケットポリシーでKnowledge Baseからの読み取りが許可されているか確認

### Lambda関数タイムアウト

**エラー**:
```
Task timed out after 15.00 seconds
```

**解決方法**:
1. Lambda関数のタイムアウト設定を確認(最大15分)
2. Bedrock APIレスポンスが遅い場合、プロンプトサイズを削減
3. ProvisionedConcurrencyを有効化してコールドスタート回避

### Cognito認証エラー

**エラー**:
```
NotAuthorizedException: Incorrect username or password
```

**解決方法**:
1. ユーザーが存在するか確認: `aws cognito-idp list-users --user-pool-id ${COGNITO_USER_POOL_ID}`
2. 初回ログイン時はパスワード変更が必要
3. ユーザーステータスが `CONFIRMED` であることを確認

---

## ログとモニタリング

### CloudWatch Logsの確認

```bash
# Lambda関数ログの確認
aws logs tail /aws/lambda/onboarding-rag-query-handler-dev --follow

# 特定の時間範囲でフィルタ
aws logs filter-log-events \
  --log-group-name /aws/lambda/onboarding-rag-query-handler-dev \
  --start-time $(date -u -d '1 hour ago' +%s)000 \
  --filter-pattern "ERROR"
```

### X-Rayトレースの確認

1. AWS Management Console → X-Ray → Traces
2. フィルタ条件でエラーや遅延リクエストを絞り込み
3. トレースマップでボトルネック特定

### メトリクスダッシュボード

CloudWatch Dashboard `OnboardingRag-dev` で以下を確認:
- API リクエスト数
- Lambda 実行時間 (p50, p95, p99)
- エラー率
- Bedrock トークン使用量

---

## 次のステップ

1. **本番環境デプロイ**: `cdk deploy --all --context env=prod`
2. **CI/CDパイプライン構築**: GitHub Actions設定
3. **負荷テスト実行**: locustで100同時ユーザーシナリオ
4. **セキュリティ監査**: IAMポリシー、暗号化設定の確認
5. **コスト最適化**: Bedrock使用量とLambdaメモリ設定の最適化

---

## 参考リンク

- [AWS CDK ドキュメント](https://docs.aws.amazon.com/cdk/)
- [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/)
- [Streamlit ドキュメント](https://docs.streamlit.io/)
- [プロジェクト仕様書](./spec.md)
- [実装計画](./plan.md)
- [データモデル](./data-model.md)
- [API仕様](./contracts/api-spec.yaml)
