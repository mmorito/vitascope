# Vitascope プロジェクトルール

このドキュメントは `docs/` ディレクトリ内の各ファイルに移行されました。
詳細な内容は [docs/README.md](docs/README.md) を参照してください。

## ドキュメント構造

プロジェクトドキュメントは肥大化を防ぐため、以下のディレクトリ構造に整理されています：

```
docs/
├── README.md (全体概要)
├── project/
│   ├── inception-deck.md (プロジェクト計画)
│   ├── design-doc.md (設計ドキュメント)
│   ├── functional-spec.md (機能仕様)
│   ├── data-model.md (データモデル)
│   ├── data-model-design.md (データモデル設計)
│   ├── fhir-data-model.md (FHIRデータモデル)
│   ├── ui-ux-design.md (UIとUXデザイン)
│   └── external-data-sources.md (外部データソース)
├── architecture/
│   ├── backend.md (DDDアーキテクチャ)
│   ├── frontend.md (FSD構造)
│   └── infrastructure.md (AWS構成)
├── development/
│   ├── api-guidelines.md (API設計ガイド)
│   ├── coding-standards.md (コード規約)
│   ├── standards/
│   │   ├── backend-coding-rule.md (バックエンド詳細規約)
│   │   └── frontend-coding-rule.md (フロントエンド詳細規約)
│   └── workflow.md (開発フロー)
└── operations/
    ├── security.md (セキュリティポリシー)
    └── monitoring.md (監視と運用)
```
