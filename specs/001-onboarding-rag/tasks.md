# タスク: 社内ドキュメントQ&Aシステム (TDDアプローチ)

**入力**: `/specs/001-onboarding-rag/` 配下の設計ドキュメント
**前提**: plan.md, spec.md, data-model.md, contracts/api-spec.yaml, research.md, quickstart.md

**開発アプローチ**: TDD(Test-Driven Development) - すべての実装前にテストを作成し、失敗することを確認してから実装する

**構成方針**: タスクはユーザーストーリーごとにグループ化し、各ストーリーを独立して実装・テストできるようにする。各ストーリー内では「テスト → 実装」の順序を厳守。

## 記法: `[ID] [P?] [Story] 説明`

- **[P]**: 並列実行可 (別ファイル、依存なし)
- **[Story]**: 所属するユーザーストーリー (例: US1, US2, US3)
- 説明には正確なファイルパスを記載

## パス規約

- **Webアプリ構成**: `backend/src/`, `frontend/src/`
- **バックエンド構成**:
  - Lambda関数: `backend/src/lambdas/`
  - サービス層: `backend/src/services/`
  - データモデル: `backend/src/models/`
  - ユーティリティ: `backend/src/utils/`
  - インフラ: `backend/infrastructure/`
  - テスト: `backend/tests/`
- **フロントエンド**: `frontend/streamlit/` (Phase 1)

---

## フェーズ1: セットアップ (共通基盤)

**目的**: プロジェクト初期化と基本構造

- [ ] T001 plan.mdに沿ってプロジェクト構造を作成 (backend/, frontend/, tests/, docs/)
- [ ] T002 [P] backend/requirements.txtに依存パッケージを定義 (boto3, langchain, pydantic等)
- [ ] T003 [P] backend/requirements-dev.txtに開発用パッケージを定義 (pytest, moto, black, ruff, mypy, pytest-cov等)
- [ ] T004 [P] frontend/streamlit/requirements.txtにStreamlit依存パッケージを定義
- [ ] T005 [P] .gitignoreファイルを作成 (Python仮想環境, AWS認証情報, .env等)
- [ ] T006 [P] backend/.envテンプレートを作成 (quickstart.mdの環境変数参考)
- [ ] T007 [P] .python-versionファイルを作成 (3.12指定)
- [ ] T008 [P] pyproject.tomlを作成 (black, ruff, pytest設定)
- [ ] T009 [P] backend/tests/conftest.pyにpytestフィクスチャを作成 (共通テスト設定)

---

## フェーズ2: 基盤構築 (すべてのユーザーストーリーの前提)

**目的**: どのユーザーストーリーにも必要な基盤を先に構築

**⚠️ 重要**: このフェーズ完了まで、ユーザーストーリーの実装に着手しない

### データモデル基盤 (TDD: テスト → 実装)

#### テスト
- [ ] T010 [P] backend/tests/unit/models/test_document.pyにDocumentモデルのユニットテストを作成
- [ ] T011 [P] backend/tests/unit/models/test_query.pyにQueryモデルのユニットテストを作成
- [ ] T012 [P] backend/tests/unit/models/test_answer.pyにAnswer, Sourceモデルのユニットテストを作成
- [ ] T013 [P] backend/tests/unit/models/test_feedback.pyにFeedbackモデルのユニットテストを作成
- [ ] T014 [P] backend/tests/unit/models/test_user.pyにUserモデルのユニットテストを作成
- [ ] T015 [P] backend/tests/unit/models/test_project.pyにProjectモデルのユニットテストを作成

#### 実装 (テスト失敗確認後)
- [ ] T016 [P] backend/src/models/document.pyにDocumentモデルを作成 (Pydantic, data-model.md参照) - T010が失敗することを確認後
- [ ] T017 [P] backend/src/models/query.pyにQueryモデルを作成 - T011が失敗することを確認後
- [ ] T018 [P] backend/src/models/answer.pyにAnswer, Sourceモデルを作成 - T012が失敗することを確認後
- [ ] T019 [P] backend/src/models/feedback.pyにFeedbackモデルを作成 - T013が失敗することを確認後
- [ ] T020 [P] backend/src/models/user.pyにUserモデルを作成 (Cognito IDトークンClaim用) - T014が失敗することを確認後
- [ ] T021 [P] backend/src/models/project.pyにProjectモデルを作成 - T015が失敗することを確認後

### ユーティリティ基盤 (TDD: テスト → 実装)

#### テスト
- [ ] T022 [P] backend/tests/unit/utils/test_logger.pyにロガーユーティリティのユニットテストを作成
- [ ] T023 [P] backend/tests/unit/utils/test_validators.pyにバリデーションユーティリティのユニットテストを作成
- [ ] T024 [P] backend/tests/unit/utils/test_exceptions.pyにカスタム例外のユニットテストを作成
- [ ] T025 [P] backend/tests/unit/utils/test_auth.pyに認証ヘルパーのユニットテストを作成

#### 実装 (テスト失敗確認後)
- [ ] T026 [P] backend/src/utils/logger.pyに構造化ログユーティリティを作成 (JSON形式, CloudWatch Logs対応) - T022が失敗することを確認後
- [ ] T027 [P] backend/src/utils/validators.pyにバリデーションユーティリティを作成 - T023が失敗することを確認後
- [ ] T028 [P] backend/src/utils/exceptions.pyにカスタム例外クラスを定義 (ValidationError, AuthorizationError等) - T024が失敗することを確認後
- [ ] T029 backend/src/utils/auth.pyに認証ヘルパーを作成 (Cognito IDトークン解析, ユーザー情報取得) - T025が失敗することを確認後

### AWS CDK インフラストラクチャ

- [ ] T030 backend/infrastructure/app.pyにCDKアプリケーションエントリーポイントを作成
- [ ] T031 [P] backend/infrastructure/stacks/storage_stack.pyにStorageStackを実装 (S3バケット, DynamoDBテーブル5つ)
- [ ] T032 [P] backend/infrastructure/stacks/bedrock_stack.pyにBedrockStackを実装 (Knowledge Base, Data Source, OpenSearch Serverless)
- [ ] T033 backend/infrastructure/stacks/api_stack.pyにApiStackを実装 (API Gateway, Lambda関数, Cognito)
- [ ] T034 backend/infrastructure/requirements.txtにCDK依存パッケージを定義 (aws-cdk-lib等)

### 基盤サービス (TDD: テスト → 実装)

#### テスト
- [ ] T035 [P] backend/tests/unit/services/test_bedrock_service.pyにBedrockServiceのユニットテストを作成
- [ ] T036 [P] backend/tests/unit/services/test_kb_service.pyにKnowledgeBaseServiceのユニットテストを作成
- [ ] T037 [P] backend/tests/unit/services/test_metadata_service.pyにMetadataServiceのユニットテストを作成
- [ ] T038 [P] backend/tests/unit/services/test_feedback_service.pyにFeedbackServiceのユニットテストを作成

#### 実装 (テスト失敗確認後)
- [ ] T039 [P] backend/src/services/bedrock_service.pyにBedrockServiceを実装 (Claude Sonnet 4.5呼び出し, プロンプトキャッシング) - T035が失敗することを確認後
- [ ] T040 [P] backend/src/services/kb_service.pyにKnowledgeBaseServiceを実装 (Retrieve API, メタデータフィルタ) - T036が失敗することを確認後
- [ ] T041 [P] backend/src/services/metadata_service.pyにMetadataServiceを実装 (S3パスからメタデータ抽出) - T037が失敗することを確認後
- [ ] T042 [P] backend/src/services/feedback_service.pyにFeedbackServiceを実装 (DynamoDB書き込み) - T038が失敗することを確認後

**チェックポイント**: 基盤完成 - ここからユーザーストーリーを並列着手可能

---

## フェーズ3: ユーザーストーリー1 - 基本的な質問応答 (優先度: P1) 🎯 MVP

**目的**: ユーザーが自然言語で質問を入力し、関連する情報と出典が即座に表示される

**独立したテスト**: Streamlit UIから質問を入力し、3秒以内に回答と出典が表示されることを確認

### ユーザーストーリー1のテスト (TDD: テスト → 実装)

#### コントラクトテスト (API仕様検証)
- [ ] T043 [P] [US1] backend/tests/contract/test_query_endpoint.pyにPOST /query エンドポイントのコントラクトテストを作成 (OpenAPI仕様準拠確認)

#### 統合テスト (ユーザージャーニー)
- [ ] T044 [P] [US1] backend/tests/integration/test_query_flow.pyに質問→回答フローの統合テストを作成 (moto使用)

#### サービス層ユニットテスト
- [ ] T045 [P] [US1] backend/tests/unit/services/test_query_service.pyにQueryServiceのユニットテストを作成
- [ ] T046 [P] [US1] backend/tests/unit/services/test_answer_service.pyにAnswerServiceのユニットテストを作成

### ユーザーストーリー1の実装 (テスト失敗確認後)

#### DynamoDBアクセス
- [ ] T047 [P] [US1] backend/src/services/query_service.pyにQueryServiceを実装 (DynamoDB QueriesテーブルへのCRUD) - T045が失敗することを確認後
- [ ] T048 [P] [US1] backend/src/services/answer_service.pyにAnswerServiceを実装 (DynamoDB AnswersテーブルへのCRUD) - T046が失敗することを確認後

#### Lambda関数 - クエリ処理
- [ ] T049 [US1] backend/src/lambdas/query_handler.pyにquery_handler Lambda関数を実装 (POST /query エンドポイント) - T043, T044が失敗することを確認後
- [ ] T050 [US1] query_handler内でユーザー認証情報を取得 (Cognito Authorizer経由)
- [ ] T051 [US1] query_handler内でリクエストバリデーションを実装 (query_textの1-2000文字チェック)
- [ ] T052 [US1] query_handler内でKnowledgeBaseService.retrieve()を呼び出し
- [ ] T053 [US1] query_handler内でBedrockService.generate_answer()を呼び出し (Claude Sonnet 4.5)
- [ ] T054 [US1] query_handler内でクエリと回答をDynamoDBに保存
- [ ] T055 [US1] query_handler内でエラーハンドリングと構造化ログを実装
- [ ] T056 [US1] query_handler内で応答時間メトリクスをCloudWatch Metricsに送信

#### Streamlit UI - 質問入力と回答表示
- [ ] T057 [US1] frontend/streamlit/app.pyにStreamlitメインアプリを作成
- [ ] T058 [US1] app.py内でCognito認証フローを実装 (ログイン画面)
- [ ] T059 [US1] app.py内で質問入力フォームを実装 (テキストエリア, 送信ボタン)
- [ ] T060 [US1] app.py内でAPI Gateway経由でPOST /queryを呼び出し
- [ ] T061 [US1] app.py内で回答テキストを表示
- [ ] T062 [US1] app.py内で参照元ドキュメント情報を表示 (ファイル名, ページ番号, 引用文, 関連度スコア)
- [ ] T063 [US1] app.py内で応答時間を表示 (3秒以内の成功基準確認用)
- [ ] T064 [US1] app.py内でエラーメッセージを表示 (該当情報なし時の通知)

#### テスト実行と検証
- [ ] T065 [US1] backend/tests/contract/test_query_endpoint.pyを実行してテストがパスすることを確認
- [ ] T066 [US1] backend/tests/integration/test_query_flow.pyを実行してテストがパスすることを確認
- [ ] T067 [US1] pytestでカバレッジレポートを生成し、US1関連コードが十分カバーされていることを確認

**チェックポイント**: ここまででユーザーストーリー1が単独で動作・テスト可能になる (Streamlitから質問 → 回答表示)

---

## フェーズ4: ユーザーストーリー2 - 案件固有情報の検索 (優先度: P2)

**目的**: ユーザーが案件を選択して質問すると、その案件固有の情報が優先的に表示される

**独立したテスト**: Streamlitで案件選択ドロップダウンから特定案件を選び、案件固有情報が回答に含まれることを確認

### ユーザーストーリー2のテスト (TDD: テスト → 実装)

#### 統合テスト (案件フィルタリング)
- [ ] T068 [P] [US2] backend/tests/integration/test_project_filtering.pyに案件選択時のフィルタリング統合テストを作成

#### サービス層ユニットテスト
- [ ] T069 [P] [US2] backend/tests/unit/services/test_project_service.pyにProjectServiceのユニットテストを作成

### ユーザーストーリー2の実装 (テスト失敗確認後)

#### 案件管理
- [ ] T070 [US2] backend/src/services/project_service.pyにProjectServiceを実装 (DynamoDB ProjectsテーブルへのCRUD) - T069が失敗することを確認後
- [ ] T071 [US2] backend/scripts/seed_projects.pyに初期プロジェクトデータ投入スクリプトを作成 (quickstart.md参照)

#### Lambda関数 - 案件フィルタリング
- [ ] T072 [US2] query_handler.pyに案件選択パラメータ(selected_projects)の処理を追加 - T068が失敗することを確認後
- [ ] T073 [US2] query_handler.py内でユーザーのアクセス可能案件を取得 (Cognito custom:projects Claim)
- [ ] T074 [US2] query_handler.py内で案件アクセス権限チェックを実装 (選択案件がユーザーのアクセス可能案件に含まれるか)
- [ ] T075 [US2] query_handler.py内でKnowledge Baseフィルタを構築 (案件選択なし → 全社共通のみ, 案件選択あり → 案件固有+全社共通)
- [ ] T076 [US2] kb_service.py内でメタデータフィルタ付き検索を実装 (research.mdの案件フィルタリング戦略参照)

#### Streamlit UI - 案件選択
- [ ] T077 [US2] app.py内で案件選択ドロップダウンを追加 (ProjectService経由でアクセス可能案件を取得)
- [ ] T078 [US2] app.py内で選択案件をPOST /queryリクエストに含める (selected_projectsパラメータ)
- [ ] T079 [US2] app.py内で案件固有情報と全社共通情報を区別して表示 (ドキュメントカテゴリ表示)

#### テスト実行と検証
- [ ] T080 [US2] backend/tests/integration/test_project_filtering.pyを実行してテストがパスすることを確認
- [ ] T081 [US2] pytestでカバレッジレポートを生成し、US2関連コードが十分カバーされていることを確認

**チェックポイント**: ここまででストーリー1と2が両方とも独立動作する (案件選択あり/なしで適切に検索)

---

## フェーズ5: ユーザーストーリー3 - フィードバックによる改善 (優先度: P3)

**目的**: ユーザーが回答の品質を評価し、コメントを残すことで、システムが継続的に改善される

**独立したテスト**: 回答表示後にフィードバックボタンをクリックし、評価とコメントがDynamoDBに保存されることを確認

### ユーザーストーリー3のテスト (TDD: テスト → 実装)

#### コントラクトテスト
- [ ] T082 [P] [US3] backend/tests/contract/test_feedback_endpoint.pyにPOST /feedback エンドポイントのコントラクトテストを作成

#### 統合テスト
- [ ] T083 [P] [US3] backend/tests/integration/test_feedback_flow.pyにフィードバック送信→保存フローの統合テストを作成

### ユーザーストーリー3の実装 (テスト失敗確認後)

#### Lambda関数 - フィードバック処理
- [ ] T084 [US3] backend/src/lambdas/feedback_handler.pyにfeedback_handler Lambda関数を実装 (POST /feedback エンドポイント) - T082, T083が失敗することを確認後
- [ ] T085 [US3] feedback_handler内でリクエストバリデーションを実装 (rating, query_id, answer_id必須チェック)
- [ ] T086 [US3] feedback_handler内で同一ユーザー・同一質問への重複フィードバックをチェック
- [ ] T087 [US3] feedback_handler内でFeedbackServiceを呼び出してDynamoDBに保存
- [ ] T088 [US3] feedback_handler内でCloudWatch Custom Metricsにフィードバックデータを送信 (満足度集計用)

#### Streamlit UI - フィードバック送信
- [ ] T089 [US3] app.py内で回答表示後にフィードバックボタンを追加 (役に立った/役に立たなかった)
- [ ] T090 [US3] app.py内でコメント入力フォームを追加 (任意, 最大1000文字)
- [ ] T091 [US3] app.py内でPOST /feedbackを呼び出し
- [ ] T092 [US3] app.py内でフィードバック送信成功メッセージを表示

#### フィードバック可視化
- [ ] T093 [US3] backend/infrastructure/stacks/monitoring_stack.pyにMonitoringStackを作成
- [ ] T094 [US3] monitoring_stack.py内でCloudWatch Dashboardを定義 (質問数/日, 満足度, 平均応答時間, エラー率)
- [ ] T095 [US3] monitoring_stack.py内でカスタムメトリクスを定義 (helpful/not_helpfulカウント)

#### テスト実行と検証
- [ ] T096 [US3] backend/tests/contract/test_feedback_endpoint.pyを実行してテストがパスすることを確認
- [ ] T097 [US3] backend/tests/integration/test_feedback_flow.pyを実行してテストがパスすることを確認
- [ ] T098 [US3] pytestでカバレッジレポートを生成し、US3関連コードが十分カバーされていることを確認

**チェックポイント**: 全ストーリーが独立動作する (質問 → 回答 → フィードバック → ダッシュボード可視化)

---

## フェーズ6: ドキュメント取り込み (管理者機能)

**目的**: 管理者がドキュメントをアップロードし、Knowledge Baseに自動同期される

**独立したテスト**: S3にドキュメントをアップロードし、Knowledge Base同期が完了してクエリで検索可能になることを確認

### ドキュメント取り込みのテスト (TDD: テスト → 実装)

#### コントラクトテスト
- [ ] T099 [P] backend/tests/contract/test_documents_endpoint.pyにPOST /documents エンドポイントのコントラクトテストを作成
- [ ] T100 [P] backend/tests/contract/test_get_documents_endpoint.pyにGET /documents エンドポイントのコントラクトテストを作成

#### E2Eテスト (主要ユーザージャーニー)
- [ ] T101 backend/tests/e2e/test_document_upload_to_search.pyにドキュメントアップロード→同期→検索のE2Eテストを作成

#### サービス層ユニットテスト
- [ ] T102 [P] backend/tests/unit/services/test_document_service.pyにDocumentServiceのユニットテストを作成

### ドキュメント取り込みの実装 (テスト失敗確認後)

#### Lambda関数 - ドキュメントアップロード
- [ ] T103 backend/src/services/document_service.pyにDocumentServiceを実装 (DynamoDB DocumentsテーブルへのCRUD) - T102が失敗することを確認後
- [ ] T104 backend/src/lambdas/document_processor.pyにdocument_processor Lambda関数を実装 (S3 Presigned URL生成, POST /documents) - T099が失敗することを確認後
- [ ] T105 document_processor内でリクエストバリデーションを実装 (file_name, file_type, categoryチェック)
- [ ] T106 document_processor内でcategoryに応じたバリデーション (common → department必須, project → project_id必須)
- [ ] T107 document_processor内でS3キーを生成 (common/{department}/, projects/{project_id}/)
- [ ] T108 document_processor内でPresigned URL生成 (有効期限5分)
- [ ] T109 document_processor内でDocumentメタデータをDynamoDBに保存 (kb_sync_status=pending)
- [ ] T110 document_processor内でGET /documents エンドポイントを実装 (ドキュメント一覧取得) - T100が失敗することを確認後

#### Lambda関数 - Knowledge Base同期
- [ ] T111 backend/src/lambdas/sync_kb.pyにsync_kb Lambda関数を実装 (S3 Event → EventBridge経由でトリガー) - T101が失敗することを確認後
- [ ] T112 sync_kb内でS3パスからメタデータを抽出 (MetadataService使用)
- [ ] T113 sync_kb内でファイル形式に応じてドキュメントをパース (PDF → Textract, DOCX → python-docx, TXT/MD → 直接読み込み)
- [ ] T114 sync_kb内でテキストをチャンク化 (langchain RecursiveCharacterTextSplitter, 500-1000文字, オーバーラップ100文字)
- [ ] T115 sync_kb内でKnowledge Base Ingestion Jobを開始 (Bedrock API)
- [ ] T116 sync_kb内でDynamoDB Documentテーブルを更新 (kb_sync_status=syncing → completed/failed)

#### EventBridge設定
- [ ] T117 backend/infrastructure/stacks/storage_stack.pyにS3 Event Notification設定を追加 (PutObject → EventBridge)
- [ ] T118 storage_stack.py内でEventBridge Ruleを作成 (S3イベント → sync_kb Lambda)

#### Streamlit UI - ドキュメント管理
- [ ] T119 [P] frontend/streamlit/components/document_upload.pyにドキュメントアップロードコンポーネントを作成
- [ ] T120 document_upload.py内でファイル選択UIを実装 (Streamlit file_uploader)
- [ ] T121 document_upload.py内でカテゴリ・部署・案件選択UIを実装
- [ ] T122 document_upload.py内でPOST /documentsを呼び出してPresigned URLを取得
- [ ] T123 document_upload.py内でPresigned URLに直接PUTリクエストでアップロード
- [ ] T124 [P] frontend/streamlit/components/document_list.pyにドキュメント一覧表示コンポーネントを作成 (GET /documents)
- [ ] T125 document_list.py内で同期ステータス表示 (pending, syncing, completed, failed)

#### テスト実行と検証
- [ ] T126 backend/tests/contract/test_documents_endpoint.pyを実行してテストがパスすることを確認
- [ ] T127 backend/tests/contract/test_get_documents_endpoint.pyを実行してテストがパスすることを確認
- [ ] T128 backend/tests/e2e/test_document_upload_to_search.pyを実行してテストがパスすることを確認
- [ ] T129 pytestでカバレッジレポートを生成し、ドキュメント取り込み関連コードが十分カバーされていることを確認

**チェックポイント**: ドキュメントアップロードから同期完了まで自動化完了

---

## フェーズ7: 質問履歴機能

**目的**: ユーザーが過去の質問と回答を確認できる

**独立したテスト**: Streamlitで質問履歴ページを開き、過去の質問が表示されることを確認

### 質問履歴のテスト (TDD: テスト → 実装)

#### コントラクトテスト
- [ ] T130 backend/tests/contract/test_query_history_endpoint.pyにGET /query-history エンドポイントのコントラクトテストを作成

### 質問履歴の実装 (テスト失敗確認後)

#### Lambda関数 - 質問履歴取得
- [ ] T131 backend/src/lambdas/query_handler.pyにGET /query-history エンドポイントを追加 - T130が失敗することを確認後
- [ ] T132 query_handler内でユーザーIDに基づいてDynamoDB QueriesテーブルからGSI1検索 (user_id-created_at-index)
- [ ] T133 query_handler内でページネーション処理を実装 (limitとnext_token)
- [ ] T134 query_handler内で各クエリに紐づく回答とフィードバックを取得 (Answersテーブル, Feedbackテーブル)

#### Streamlit UI - 質問履歴表示
- [ ] T135 [P] frontend/streamlit/components/query_history.pyに質問履歴コンポーネントを作成
- [ ] T136 query_history.py内でGET /query-historyを呼び出し
- [ ] T137 query_history.py内で質問・回答・フィードバックをリスト形式で表示
- [ ] T138 query_history.py内でページネーション機能を実装 (次へ/前へボタン)

#### テスト実行と検証
- [ ] T139 backend/tests/contract/test_query_history_endpoint.pyを実行してテストがパスすることを確認

**チェックポイント**: 質問履歴機能が完全に動作

---

## フェーズ8: E2Eテストと負荷テスト

**目的**: 主要ユーザージャーニー全体の動作確認と性能検証

### E2Eテスト

- [ ] T140 backend/tests/e2e/test_full_user_journey.pyに完全なユーザージャーニーのE2Eテストを作成 (質問→回答→フィードバック)
- [ ] T141 backend/tests/e2e/conftest.pyにE2Eテスト用フィクスチャを作成 (実AWS環境接続)
- [ ] T142 backend/tests/e2e/test_full_user_journey.pyを実行してテストがパスすることを確認

### 負荷テスト

- [ ] T143 backend/tests/load/locustfile.pyに負荷テストシナリオを作成 (100同時ユーザー, POST /query)
- [ ] T144 locustで負荷テストを実行し、応答時間3秒以内(p95)を維持することを確認 (SC-001, SC-005)

---

## フェーズ9: 仕上げと横断的対応

**目的**: 複数ストーリーにまたがる改善とデプロイ準備

### デプロイとテスト

- [ ] T145 quickstart.mdの手順に従ってローカル開発環境をセットアップ
- [ ] T146 CDKで開発環境にデプロイ (cdk deploy --all --context env=dev)
- [ ] T147 Bedrock モデルアクセスを有効化 (Claude Sonnet 4.5, Titan Embeddings V2)
- [ ] T148 Cognitoユーザーを作成してStreamlit UIからログインテスト
- [ ] T149 テストドキュメントをS3にアップロードしてKnowledge Base同期を確認
- [ ] T150 Streamlitから質問を実行し、3秒以内に回答が表示されることを確認 (SC-001)
- [ ] T151 フィードバック機能を実行し、CloudWatch Dashboardで可視化されることを確認

### カバレッジ検証

- [ ] T152 pytest --cov=backend/src --cov-report=html を実行してカバレッジレポートを生成
- [ ] T153 カバレッジ率が80%以上であることを確認 (TR-006)
- [ ] T154 カバレッジが不足している箇所に追加のユニットテストを作成

### ドキュメント整備

- [ ] T155 [P] docs/architecture.mdにアーキテクチャ図と説明を作成
- [ ] T156 [P] docs/deployment.mdにデプロイ手順を作成 (本番環境向け)
- [ ] T157 [P] README.mdにプロジェクト概要とクイックスタートリンクを作成
- [ ] T158 [P] CLAUDE.mdを更新 (最新の技術スタックと構成を反映)

### コード品質改善

- [ ] T159 backend全体でblackフォーマッタを実行
- [ ] T160 backend全体でruff linterを実行してワーニングを修正
- [ ] T161 backend全体でmypyの型チェックを実行

### セキュリティとパフォーマンス

- [ ] T162 Lambda関数のIAMロール最小権限を確認 (least privilege原則)
- [ ] T163 S3バケットの暗号化設定を確認 (SSE-S3)
- [ ] T164 DynamoDBテーブルの暗号化設定を確認
- [ ] T165 CloudWatch LogsからPII情報をマスキングする設定を追加
- [ ] T166 Lambda Power Tuningで最適メモリ設定を測定 (research.md参照)
- [ ] T167 query_handler LambdaにProvisionedConcurrencyを設定 (コールドスタート回避)

### CI/CDパイプライン (オプション)

- [ ] T168 [P] .github/workflows/ci.ymlにCI/CDパイプラインを作成 (GitHub Actions)
- [ ] T169 ci.yml内でPythonテスト実行ステップを追加 (pytest, lint, type-check)
- [ ] T170 ci.yml内でCDK deployステップを追加 (dev環境自動デプロイ)

---

## 依存関係と実行順序

### フェーズ間の依存

- **セットアップ (フェーズ1)**: 依存なし - すぐ着手可
- **基盤構築 (フェーズ2)**: セットアップ完了後 - 全ストーリーの前提
- **ユーザーストーリー1 (フェーズ3)**: 基盤構築完了後
- **ユーザーストーリー2 (フェーズ4)**: 基盤構築完了後、US1完成後推奨
- **ユーザーストーリー3 (フェーズ5)**: 基盤構築完了後、US1完成後推奨
- **ドキュメント取り込み (フェーズ6)**: 基盤構築完了後、US1と並行可能
- **質問履歴 (フェーズ7)**: US1完成後
- **E2E・負荷テスト (フェーズ8)**: US1-3完成後
- **仕上げ (フェーズ9)**: 必要なストーリー完成後

### TDD サイクル (各ストーリー内)

**重要**: 各ユーザーストーリー内では必ず以下の順序を守る

1. **テスト作成**: コントラクトテスト、統合テスト、ユニットテストを先に作成
2. **テスト失敗確認**: pytest を実行し、すべてのテストが失敗することを確認 (Red)
3. **実装**: 失敗したテストをパスさせるために最小限の実装を追加 (Green)
4. **リファクタリング**: テストがパスする状態を維持しながらコードを改善 (Refactor)
5. **テスト成功確認**: pytest を実行し、すべてのテストがパスすることを確認

### 並列実行の機会

#### フェーズ1 (セットアップ)
- T002-T009 を並列実行可能

#### フェーズ2 (基盤構築)
- データモデルテスト (T010-T015) を並列実行可能
- データモデル実装 (T016-T021) を並列実行可能 (対応するテスト失敗確認後)
- ユーティリティテスト (T022-T025) を並列実行可能
- ユーティリティ実装 (T026-T029) を並列実行可能 (対応するテスト失敗確認後)
- インフラスタック (T031, T032) を並列実行可能
- 基盤サービステスト (T035-T038) を並列実行可能
- 基盤サービス実装 (T039-T042) を並列実行可能 (対応するテスト失敗確認後)

#### フェーズ3 (US1)
- コントラクトテスト・統合テスト (T043-T044) を並列実行可能
- サービス層ユニットテスト (T045-T046) を並列実行可能
- サービス層実装 (T047-T048) を並列実行可能 (対応するテスト失敗確認後)

#### フェーズ6 (ドキュメント取り込み)
- コントラクトテスト (T099-T100) を並列実行可能
- UI コンポーネント (T119, T124) を並列実行可能

#### フェーズ9 (仕上げ)
- ドキュメント整備 (T155-T158) を並列実行可能
- T168-T170 (CI/CD) を並列実行可能

---

## 並列実行の例: ユーザーストーリー1 (TDDサイクル)

```bash
# ステップ1: テストを並列で作成
Task: "backend/tests/contract/test_query_endpoint.pyにPOST /query エンドポイントのコントラクトテストを作成"
Task: "backend/tests/integration/test_query_flow.pyに質問→回答フローの統合テストを作成"
Task: "backend/tests/unit/services/test_query_service.pyにQueryServiceのユニットテストを作成"
Task: "backend/tests/unit/services/test_answer_service.pyにAnswerServiceのユニットテストを作成"

# ステップ2: テスト失敗確認
pytest backend/tests/  # すべて失敗することを確認 (Red)

# ステップ3: 実装を並列で実施
Task: "backend/src/services/query_service.pyにQueryServiceを実装"
Task: "backend/src/services/answer_service.pyにAnswerServiceを実装"

# ステップ4: テスト成功確認
pytest backend/tests/  # すべてパスすることを確認 (Green)
```

---

## 実装の進め方 (TDDアプローチ)

### MVPファースト (ストーリー1のみ)

1. フェーズ1: セットアップを完了
2. フェーズ2: 基盤構築を完了 (TDDサイクル: テスト → 失敗確認 → 実装 → 成功確認)
3. フェーズ3: ユーザーストーリー1を完了 (TDDサイクル厳守)
4. **停止して検証**:
   - すべてのテストがパスすることを確認
   - カバレッジ80%以上を確認
   - ストーリー1を手動で単独テスト (質問 → 回答 → 3秒以内)
5. 準備できたらデプロイ・デモ (MVP完成!)

### TDDサイクルの実践

各ユーザーストーリーで以下を繰り返す:

1. **Red**: テストを書いて失敗させる
   - コントラクトテスト、統合テスト、ユニットテストを先に作成
   - pytest を実行して失敗することを確認
2. **Green**: 最小限の実装でテストを通す
   - 失敗したテストをパスさせるコードを書く
   - pytest を実行してパスすることを確認
3. **Refactor**: コードを改善
   - テストがパスする状態を維持しながらリファクタリング
   - pytest を再実行してパスすることを確認

### 段階的リリース

1. セットアップ + 基盤構築 → 基盤完成 (すべてのテストパス)
2. ストーリー1追加 → すべてのテストパス → デプロイ・デモ (MVP!)
3. ストーリー2追加 → すべてのテストパス → デプロイ・デモ
4. ストーリー3追加 → すべてのテストパス → デプロイ・デモ
5. ドキュメント取り込み追加 → E2Eテストパス → デプロイ・デモ
6. 質問履歴追加 → すべてのテストパス → デプロイ・デモ (完全版!)

### チーム並列開発 (TDD維持)

複数の開発者がいる場合:

1. チームでセットアップ + 基盤構築を完了 (TDDサイクル厳守)
2. 基盤完了後:
   - 開発者A: ストーリー1 (TDDサイクル)
   - 開発者B: ストーリー2 (TDDサイクル) ※US1完成後推奨
   - 開発者C: ドキュメント取り込み (TDDサイクル)
3. 各ストーリーを独立完成・統合 (すべてのテストパス確認)

---

## 補足

- **TDD必須**: すべての実装前にテストを作成し、失敗することを確認してから実装
- [P]タスク = 別ファイル、依存なし
- [Story]ラベルでタスクとストーリーを紐付け、トレーサビリティ確保
- 各ストーリーは独立完成・テスト可能にする
- 各タスクまたは論理グループごとにコミット
- 各チェックポイントで停止し、すべてのテストがパスすることを確認
- カバレッジ80%以上を常に維持
- 避けるべきこと: テストなしの実装、曖昧なタスク、同一ファイルの競合、ストーリー間の強依存

---

## タスクサマリー

**総タスク数**: 170タスク

### ユーザーストーリー別タスク数
- **セットアップ (フェーズ1)**: 9タスク
- **基盤構築 (フェーズ2)**: 33タスク (テスト17 + 実装16)
- **ユーザーストーリー1 (P1, MVP)**: 25タスク (テスト6 + 実装18 + 検証1)
- **ユーザーストーリー2 (P2)**: 14タスク (テスト2 + 実装10 + 検証2)
- **ユーザーストーリー3 (P3)**: 17タスク (テスト2 + 実装12 + 検証3)
- **ドキュメント取り込み (フェーズ6)**: 31タスク (テスト5 + 実装23 + 検証3)
- **質問履歴 (フェーズ7)**: 10タスク (テスト1 + 実装8 + 検証1)
- **E2E・負荷テスト (フェーズ8)**: 5タスク
- **仕上げ (フェーズ9)**: 26タスク

### テスト関連タスク数
- **ユニットテスト**: 約35タスク
- **統合テスト**: 約8タスク
- **コントラクトテスト**: 約7タスク
- **E2Eテスト**: 約4タスク
- **負荷テスト**: 2タスク
- **テスト検証**: 約10タスク

**テストタスク合計**: 約66タスク (全体の約39%)

### 並列実行可能タスク
- **[P]マーク付きタスク**: 約60タスク (全体の約35%)
- **独立して並列実行可能なフェーズ**: フェーズ2の基盤構築、フェーズ3-6のユーザーストーリー (基盤完成後)

### 推奨MVPスコープ (TDD完全適用)
- **フェーズ1**: セットアップ (9タスク)
- **フェーズ2**: 基盤構築 (33タスク - テスト → 実装)
- **フェーズ3**: ユーザーストーリー1 (25タスク - テスト → 実装)

**MVP合計**: 67タスク (テスト含む)

### テスト要件達成確認

- ✅ **TR-001**: TDDアプローチ - すべての実装前にテスト作成
- ✅ **TR-002**: コントラクトテスト - 全APIエンドポイントにコントラクトテスト
- ✅ **TR-003**: 統合テスト - 各ユーザーストーリーの受け入れ条件に対応
- ✅ **TR-004**: E2Eテスト - 主要ユーザージャーニーをカバー
- ✅ **TR-005**: ユニットテスト - すべてのサービス層にユニットテスト
- ✅ **TR-006**: カバレッジ80%以上 - T152-T154で検証
- ✅ **TR-007**: 負荷テスト - T143-T144で100同時ユーザー検証

### 独立したテスト基準

#### ユーザーストーリー1
- ✅ すべてのユニットテストがパス (QueryService, AnswerService)
- ✅ コントラクトテストがパス (POST /query)
- ✅ 統合テストがパス (質問→回答フロー)
- ✅ Streamlit UIから「経費精算の方法は?」と質問
- ✅ 3秒以内に回答が表示される
- ✅ 回答に情報源(ドキュメント名、ページ番号)が明示される

#### ユーザーストーリー2
- ✅ 統合テストがパス (案件フィルタリング)
- ✅ 案件選択ドロップダウンから「project-001」を選択
- ✅ 「デプロイ手順は?」と質問
- ✅ 案件固有のデプロイ手順が回答に含まれる
- ✅ 全社共通情報も参照できる

#### ユーザーストーリー3
- ✅ コントラクトテストがパス (POST /feedback)
- ✅ 統合テストがパス (フィードバック送信→保存フロー)
- ✅ 回答表示後に「役に立った」ボタンをクリック
- ✅ コメント欄に「とても参考になりました」と入力
- ✅ フィードバック送信成功メッセージが表示される
- ✅ CloudWatch Dashboardで満足度が集計される
