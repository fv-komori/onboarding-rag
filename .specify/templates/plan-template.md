# 実装計画: [機能]

**ブランチ**: `[###-feature-name]` | **日付**: [DATE] | **仕様**: [link]
**入力**: `/specs/[###-feature-name]/spec.md`の機能仕様

**注**: このテンプレートは`/speckit.plan`コマンドで埋められる。実行フローは`.specify/templates/commands/plan.md`を参照。

## 概要

[機能仕様から抽出: 主要要件 + 調査で決めた技術アプローチ]

## 技術スタック

<!--
  要対応: プロジェクトの技術構成に置き換えること
  この構造はガイドとして示している
-->

**言語/バージョン**: [例: Python 3.11, Swift 5.9, Rust 1.75 または 要明確化]
**主な依存ライブラリ**: [例: FastAPI, UIKit, LLVM または 要明確化]
**データ保存**: [該当する場合、例: PostgreSQL, CoreData, files または N/A]
**テストツール**: [例: pytest, XCTest, cargo test または 要明確化]
**実行環境**: [例: Linux server, iOS 15+, WASM または 要明確化]
**プロジェクト種別**: [single/web/mobile - ソース構成を決定]
**性能目標**: [ドメイン固有、例: 1000 req/s, 10k lines/sec, 60 fps または 要明確化]
**制約事項**: [ドメイン固有、例: <200ms p95, <100MB memory, offline-capable または 要明確化]
**規模感**: [ドメイン固有、例: 10k users, 1M LOC, 50 screens または 要明確化]

## 憲法チェック

*ゲート条件: Phase 0調査の前に通過必須。Phase 1設計後に再チェック。*

[憲法ファイルに基づいて決定]

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
<!--
  要対応: 下記のプレースホルダーを実際の構成に置き換えること
  使わないオプションは削除し、選んだ構成を実際のパス(例: apps/admin, packages/something)で展開する
  最終版にはOptionラベルを含めないこと
-->

```text
# [未使用なら削除] オプション1: 単一プロジェクト (デフォルト)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [未使用なら削除] オプション2: Webアプリケーション ("frontend" + "backend"が検出された場合)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [未使用なら削除] オプション3: モバイル + API ("iOS/Android"が検出された場合)
api/
└── [上記のbackendと同様]

ios/ または android/
└── [プラットフォーム固有: 機能モジュール, UIフロー, プラットフォームテスト]
```

**構成の決定**: [選んだ構成を記述し、上記の実ディレクトリを参照]

## 複雑度の管理

> **憲法チェックで違反があり正当化が必要な場合のみ記入**

| 違反内容 | なぜ必要か | よりシンプルな代替案を採用しない理由 |
|---------|-----------|--------------------------------|
| [例: 4つ目のプロジェクト] | [現在の要件] | [3つのプロジェクトでは対応できない理由] |
| [例: Repositoryパターン] | [具体的な課題] | [直接DBアクセスでは不十分な理由] |
