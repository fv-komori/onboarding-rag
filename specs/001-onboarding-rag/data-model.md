# データモデル: 社内ドキュメントQ&Aシステム

**ブランチ**: `001-onboarding-rag` | **日付**: 2025-12-01

## 概要

このドキュメントは、システムで扱う主要なエンティティとその関係性を定義する。各エンティティはバックエンド(Lambda関数、DynamoDB)とフロントエンド(Streamlit/React)で共有される。

---

## エンティティ一覧

1. **Document**: ドキュメントメタデータ
2. **Query**: ユーザーの質問
3. **Answer**: LLMが生成した回答
4. **Feedback**: ユーザーフィードバック
5. **User**: ユーザー情報(Cognito管理)
6. **Project**: 案件情報

---

## エンティティ詳細

### 1. Document

**説明**: アップロードされたドキュメントのメタデータ。実体はS3に保存され、Knowledge Baseにインデックス化される。

**フィールド**:

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `document_id` | String (UUID) | ✅ | ドキュメントの一意ID | UUID v4形式 |
| `s3_key` | String | ✅ | S3オブジェクトキー | 例: `common/hr/就業規則.pdf` |
| `s3_bucket` | String | ✅ | S3バケット名 | 英数字とハイフン |
| `file_name` | String | ✅ | 元のファイル名 | 1-255文字 |
| `file_type` | Enum | ✅ | ファイル形式 | `pdf`, `docx`, `txt`, `md` |
| `file_size` | Integer | ✅ | ファイルサイズ(バイト) | > 0 |
| `category` | Enum | ✅ | カテゴリ | `common`, `project` |
| `department` | String | ❌ | 部署 (categoryがcommonの場合) | 例: `hr`, `finance` |
| `project_id` | String | ❌ | 案件ID (categoryがprojectの場合) | 例: `project-001` |
| `uploaded_at` | ISO8601 Timestamp | ✅ | アップロード日時 | ISO8601形式 |
| `uploaded_by` | String (User ID) | ✅ | アップロードしたユーザーID | Cognito User Sub |
| `kb_sync_status` | Enum | ✅ | Knowledge Base同期ステータス | `pending`, `syncing`, `completed`, `failed` |
| `kb_sync_at` | ISO8601 Timestamp | ❌ | 同期完了日時 | ISO8601形式 |
| `chunk_count` | Integer | ❌ | チャンク数 | ≥ 0 |
| `metadata` | Object | ❌ | 追加メタデータ | JSON Object |

**状態遷移**:
```
uploaded (kb_sync_status=pending)
  → syncing (Knowledge Base同期中)
  → completed (同期完了、検索可能)
  → failed (同期失敗、エラーログ記録)
```

**DynamoDBテーブル設計**:
- **テーブル名**: `Documents`
- **パーティションキー**: `document_id`
- **GSI1**: `category-uploaded_at-index` (カテゴリ別一覧取得)
- **GSI2**: `project_id-uploaded_at-index` (案件別ドキュメント取得)

**Pythonモデル例** (Pydantic):
```python
from pydantic import BaseModel, Field
from typing import Optional, Literal
from datetime import datetime
from uuid import UUID

class Document(BaseModel):
    document_id: UUID
    s3_key: str = Field(..., min_length=1, max_length=1024)
    s3_bucket: str
    file_name: str = Field(..., min_length=1, max_length=255)
    file_type: Literal["pdf", "docx", "txt", "md"]
    file_size: int = Field(..., gt=0)
    category: Literal["common", "project"]
    department: Optional[str] = None
    project_id: Optional[str] = None
    uploaded_at: datetime
    uploaded_by: str
    kb_sync_status: Literal["pending", "syncing", "completed", "failed"]
    kb_sync_at: Optional[datetime] = None
    chunk_count: Optional[int] = None
    metadata: Optional[dict] = None
```

---

### 2. Query

**説明**: ユーザーが入力した質問。回答と紐付けられる。

**フィールド**:

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `query_id` | String (UUID) | ✅ | 質問の一意ID | UUID v4形式 |
| `user_id` | String | ✅ | 質問者のユーザーID | Cognito User Sub |
| `query_text` | String | ✅ | 質問文 | 1-2000文字 |
| `selected_projects` | Array[String] | ❌ | 選択された案件IDリスト | 例: `["project-001"]` |
| `created_at` | ISO8601 Timestamp | ✅ | 質問日時 | ISO8601形式 |
| `answer_id` | String (UUID) | ❌ | 紐付く回答ID | UUID v4形式 |
| `session_id` | String (UUID) | ❌ | セッションID(会話履歴管理用) | UUID v4形式 |

**DynamoDBテーブル設計**:
- **テーブル名**: `Queries`
- **パーティションキー**: `query_id`
- **GSI1**: `user_id-created_at-index` (ユーザー別質問履歴)
- **GSI2**: `session_id-created_at-index` (セッション別質問履歴)

**Pythonモデル例**:
```python
class Query(BaseModel):
    query_id: UUID
    user_id: str
    query_text: str = Field(..., min_length=1, max_length=2000)
    selected_projects: Optional[list[str]] = None
    created_at: datetime
    answer_id: Optional[UUID] = None
    session_id: Optional[UUID] = None
```

---

### 3. Answer

**説明**: LLMが生成した回答と検索結果。

**フィールド**:

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `answer_id` | String (UUID) | ✅ | 回答の一意ID | UUID v4形式 |
| `query_id` | String (UUID) | ✅ | 紐付く質問ID | UUID v4形式 |
| `answer_text` | String | ✅ | 回答テキスト | 1-10000文字 |
| `sources` | Array[Source] | ✅ | 参照元ドキュメント情報 | 1-10個 |
| `created_at` | ISO8601 Timestamp | ✅ | 回答生成日時 | ISO8601形式 |
| `generation_time_ms` | Integer | ✅ | 生成時間(ミリ秒) | > 0 |
| `model_id` | String | ✅ | 使用したLLMモデルID | 例: `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| `token_count` | Object | ✅ | トークン使用量 | `{input: int, output: int}` |
| `kb_search_time_ms` | Integer | ✅ | Knowledge Base検索時間(ミリ秒) | > 0 |

**サブエンティティ: Source**

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `document_id` | String (UUID) | ✅ | 参照元ドキュメントID | UUID v4形式 |
| `file_name` | String | ✅ | ファイル名 | 表示用 |
| `s3_key` | String | ✅ | S3オブジェクトキー | ダウンロード用 |
| `page_number` | Integer | ❌ | ページ番号(PDFの場合) | > 0 |
| `chunk_text` | String | ✅ | 抽出されたチャンク(引用文) | 1-1000文字 |
| `relevance_score` | Float | ✅ | 関連度スコア | 0.0-1.0 |

**DynamoDBテーブル設計**:
- **テーブル名**: `Answers`
- **パーティションキー**: `answer_id`
- **GSI1**: `query_id-index` (質問から回答を取得)

**Pythonモデル例**:
```python
class Source(BaseModel):
    document_id: UUID
    file_name: str
    s3_key: str
    page_number: Optional[int] = None
    chunk_text: str = Field(..., min_length=1, max_length=1000)
    relevance_score: float = Field(..., ge=0.0, le=1.0)

class Answer(BaseModel):
    answer_id: UUID
    query_id: UUID
    answer_text: str = Field(..., min_length=1, max_length=10000)
    sources: list[Source] = Field(..., min_items=1, max_items=10)
    created_at: datetime
    generation_time_ms: int = Field(..., gt=0)
    model_id: str
    token_count: dict[str, int]  # {"input": 1000, "output": 500}
    kb_search_time_ms: int = Field(..., gt=0)
```

---

### 4. Feedback

**説明**: ユーザーが回答に対して送信したフィードバック。

**フィールド**:

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `feedback_id` | String (UUID) | ✅ | フィードバックの一意ID | UUID v4形式 |
| `query_id` | String (UUID) | ✅ | 紐付く質問ID | UUID v4形式 |
| `answer_id` | String (UUID) | ✅ | 紐付く回答ID | UUID v4形式 |
| `user_id` | String | ✅ | フィードバック送信者のユーザーID | Cognito User Sub |
| `rating` | Enum | ✅ | 評価 | `helpful`, `not_helpful` |
| `comment` | String | ❌ | コメント(任意) | 0-1000文字 |
| `created_at` | ISO8601 Timestamp | ✅ | フィードバック日時 | ISO8601形式 |

**DynamoDBテーブル設計**:
- **テーブル名**: `Feedback`
- **パーティションキー**: `feedback_id`
- **GSI1**: `query_id-created_at-index` (質問別フィードバック)
- **GSI2**: `user_id-created_at-index` (ユーザー別フィードバック履歴)

**Pythonモデル例**:
```python
class Feedback(BaseModel):
    feedback_id: UUID
    query_id: UUID
    answer_id: UUID
    user_id: str
    rating: Literal["helpful", "not_helpful"]
    comment: Optional[str] = Field(None, max_length=1000)
    created_at: datetime
```

---

### 5. User

**説明**: ユーザー情報。Amazon Cognitoで管理されるため、DynamoDBには保存しない。必要に応じてIDトークンClaimから取得。

**フィールド** (Cognito IDトークンClaim):

| フィールド名 | 型 | 説明 |
|-------------|-----|------|
| `sub` | String (UUID) | Cognito User ID (一意識別子) |
| `email` | String | メールアドレス |
| `name` | String | 表示名 |
| `custom:department` | String | 部署 (カスタム属性) |
| `custom:projects` | String | アクセス可能案件IDリスト(カンマ区切り) |

**Pythonモデル例**:
```python
class User(BaseModel):
    sub: str
    email: str
    name: str
    department: Optional[str] = None
    projects: Optional[list[str]] = None  # custom:projectsをパース
```

---

### 6. Project

**説明**: 案件情報。初期実装では静的な設定ファイルまたはDynamoDBで管理。

**フィールド**:

| フィールド名 | 型 | 必須 | 説明 | バリデーション |
|-------------|-----|------|------|---------------|
| `project_id` | String | ✅ | 案件の一意ID | 例: `project-001` |
| `project_name` | String | ✅ | 案件名 | 1-255文字 |
| `description` | String | ❌ | 案件説明 | 0-1000文字 |
| `created_at` | ISO8601 Timestamp | ✅ | 作成日時 | ISO8601形式 |
| `is_active` | Boolean | ✅ | アクティブフラグ | true/false |

**DynamoDBテーブル設計**:
- **テーブル名**: `Projects`
- **パーティションキー**: `project_id`

**Pythonモデル例**:
```python
class Project(BaseModel):
    project_id: str = Field(..., min_length=1, max_length=100)
    project_name: str = Field(..., min_length=1, max_length=255)
    description: Optional[str] = Field(None, max_length=1000)
    created_at: datetime
    is_active: bool = True
```

---

## エンティティ関係図

```
User (Cognito)
  │
  ├─── 1:N ──→ Query
  │             │
  │             └─── 1:1 ──→ Answer
  │                           │
  │                           └─── N:M ──→ Document (via sources)
  │
  ├─── 1:N ──→ Feedback
  │             │
  │             └─── N:1 ──→ Query
  │
  └─── N:M ──→ Project (custom:projects Claim)

Document
  │
  └─── N:1 ──→ Project (project_id, categoryがprojectの場合)

Project
  │
  └─── 1:N ──→ Document
```

**関係性の説明**:
- **User - Query**: 1ユーザーは複数の質問を作成(1:N)
- **Query - Answer**: 1質問に対して1回答(1:1)
- **Answer - Document**: 1回答は複数のドキュメントを参照(N:M、`sources`経由)
- **User - Feedback**: 1ユーザーは複数のフィードバックを送信(1:N)
- **Feedback - Query**: 複数のフィードバックが1質問に紐付く可能性(N:1)
- **User - Project**: ユーザーは複数の案件にアクセス可能(N:M、Cognito Claimで管理)
- **Document - Project**: 1ドキュメントは1案件に所属(N:1、`category=project`の場合)

---

## バリデーションルール

### ドメインロジックバリデーション

1. **Document**:
   - `category=common`の場合、`department`は必須、`project_id`はnull
   - `category=project`の場合、`project_id`は必須、`department`はnull
   - `kb_sync_status=completed`の場合、`kb_sync_at`と`chunk_count`は必須

2. **Query**:
   - `selected_projects`が空またはnullの場合、全社共通ドキュメントのみ検索
   - `selected_projects`に含まれる案件IDは、ユーザーの`custom:projects` Claimに含まれる必要がある(認可チェック)

3. **Answer**:
   - `sources`は最低1個、最大10個
   - `sources`内の各`relevance_score`は降順にソート

4. **Feedback**:
   - `feedback_id`は`query_id`と`user_id`の組み合わせでユニーク(同じユーザーが同じ質問に複数回フィードバック不可)

---

## DynamoDBテーブル設計まとめ

| テーブル名 | パーティションキー | ソートキー | GSI1 | GSI2 |
|-----------|------------------|----------|------|------|
| `Documents` | `document_id` | - | `category-uploaded_at-index` | `project_id-uploaded_at-index` |
| `Queries` | `query_id` | - | `user_id-created_at-index` | `session_id-created_at-index` |
| `Answers` | `answer_id` | - | `query_id-index` | - |
| `Feedback` | `feedback_id` | - | `query_id-created_at-index` | `user_id-created_at-index` |
| `Projects` | `project_id` | - | - | - |

**アクセスパターン**:
1. ユーザー別質問履歴取得: GSI1 on `Queries` (`user_id-created_at-index`)
2. 案件別ドキュメント一覧: GSI2 on `Documents` (`project_id-uploaded_at-index`)
3. 質問から回答取得: GSI1 on `Answers` (`query_id-index`)
4. 質問別フィードバック一覧: GSI1 on `Feedback` (`query_id-created_at-index`)

---

## S3データ構造

**バケット**: `{project-name}-documents-{env}`

**オブジェクトキー構造**:
```
common/
├── hr/
│   ├── 就業規則_v1.pdf
│   └── 休暇申請ガイド.docx
├── finance/
│   └── 経費精算ルール.pdf
└── it/
    └── システム利用ガイド.md

projects/
├── project-001/
│   ├── マニュアル.pdf
│   └── ユビキタス言語.md
└── project-002/
    └── 設計書.docx
```

**メタデータ**: S3オブジェクトタグでカテゴリ、部署、案件IDを付与(Lambda処理時)

---

**次のステップ**: Phase 1のcontracts/ (API仕様)生成に進む。
