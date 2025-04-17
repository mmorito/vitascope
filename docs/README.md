# Vitascope プロジェクトドキュメント

このディレクトリには、Vitascope プロジェクトの開発・運用に関するドキュメントが含まれています。

## プロジェクト概要

Vitascope は、社会生活者のライフメトリクスと医療ログを統合し、新たな医療インサイトを提供するプラットフォームです。このリポジトリは PHR（Personal Health Record）アプリの開発・管理を行います。

## プロジェクト計画

- [インセプションデッキ](./project/inception-deck.md)
- [デザインドキュメント](./project/design-doc.md)
- [FHIR データモデル設計](./project/fhir-data-model.md)
- [機能仕様](./project/functional-spec.md)
- [データモデル設計](./project/data-model-design.md)
- [UI と UX デザイン](./project/ui-ux-design.md)
- [外部データソース](./project/external-data-sources.md)

## ドキュメント構造

### アーキテクチャ

- [バックエンドアーキテクチャ (DDD)](./architecture/backend.md)
- [フロントエンドアーキテクチャ (FSD)](./architecture/frontend.md)
- [インフラストラクチャ](./architecture/infrastructure.md)

### 開発ガイドライン

- [API 設計ガイドライン](./development/api-guidelines.md)
- [コーディング規約](./development/coding-standards.md)
- [開発フロー](./development/workflow.md)
- [バックエンドコーディングルール](./development/standards/backend-coding-rule.md)
- [フロントエンドコーディングルール](./development/standards/frontend-coding-rule.md)

### 運用

- [セキュリティ](./operations/security.md)
- [監視とアラート](./operations/monitoring.md)

## 主要原則

### 1. ユーザー中心設計

- ユーザー（患者、医療従事者）のニーズを常に最優先
- アクセシビリティとユーザビリティを重視
- 定期的なユーザーテストとフィードバックの収集

### 2. データセキュリティとプライバシー

- HIPAA、GDPR など関連法規制の厳守
- ゼロトラスト原則に基づいたセキュリティ設計
- 最小限のデータ収集とデータの匿名化
- エンドツーエンド暗号化の実装

### 3. スケーラビリティと拡張性

- マイクロサービスアーキテクチャの採用
- クラウドネイティブな設計
- API 駆動開発

### 4. 品質と信頼性

- テスト駆動開発（TDD）の実践
- 継続的インテグレーション/継続的デリバリー（CI/CD）
- 包括的な自動テスト（単体、統合、E2E）
- 徹底したコードレビュー
