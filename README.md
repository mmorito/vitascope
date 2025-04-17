# Vitascope

## 概要

Vitascope は、社会生活者のライフメトリクスと医療ログを統合し、新たな医療インサイトを提供するためのプラットフォームです。このリポジトリでは PHR（Personal Health Record）アプリの開発を管理しています。

## ビジョン

すべての人々が自分の健康データを完全に把握し、医療機関とシームレスに連携できる世界を実現します。生活データと医療データの融合により、予防医療の促進、個別化医療の実現、希少疾患の解明に貢献します。

## 主な機能

- 日常生活メトリクスの自動収集（活動量、睡眠、栄養など）
- 医療記録の一元管理と可視化
- 健康トレンド分析とアラート
- 医療機関との安全なデータ共有
- 個人に最適化された健康アドバイス

## 技術スタック

### バックエンド

- **言語**: Python 3.9+
- **フレームワーク**: FastAPI
- **ORM**: SQLModel
- **データベース**: PostgreSQL（JSONB 対応）
- **認証**: Amazon Cognito

### フロントエンド

- **フレームワーク**: React Native
- **開発環境**: Expo
- **状態管理**: Redux Toolkit
- **スタイリング**: NativeWind

### インフラ

- **クラウド**: AWS
- **コンテナ**: Docker, Kubernetes
- **CI/CD**: GitHub Actions

## 開発環境のセットアップ

1. 必要条件

   - Node.js (LTS)
   - Python 3.9+
   - Docker
   - Git

2. インストール

   ```bash
   # リポジトリのクローン
   git clone https://github.com/yourusername/vitascope.git
   cd vitascope

   # 依存関係のインストール
   npm install

   # 環境変数の設定
   cp .env.example .env
   # .envファイルを適切に編集してください

   # 開発サーバーの起動
   npm run dev
   ```

## プロジェクト構造

```
vitascope/
├── app/                  # モバイルアプリケーションコード
│   ├── src/              # ソースコード
│   │   ├── app/          # アプリケーションロジック
│   │   ├── entities/     # ドメインエンティティ (FSD)
│   │   ├── features/     # 機能モジュール (FSD)
│   │   ├── pages/        # 画面コンポーネント (FSD)
│   │   ├── shared/       # 共通コンポーネント (FSD)
│   │   └── widgets/      # 複合コンポーネント (FSD)
│   ├── assets/           # 静的ファイル
│   └── tests/            # テスト
├── server/               # バックエンドコード
│   ├── src/              # ソースコード
│   │   ├── api/          # APIエンドポイント
│   │   ├── domain/       # ドメインロジック (DDD)
│   │   ├── application/  # アプリケーションサービス (DDD)
│   │   ├── infrastructure/ # インフラ層 (DDD)
│   │   └── utils/        # ユーティリティ
│   └── tests/            # テスト
├── docs/                 # プロジェクトドキュメント
│   └── README.md         # ドキュメント目次
└── infrastructure/       # インフラストラクチャコード
    └── terraform/        # IaC (Terraform)
```

## ドキュメント

プロジェクトの詳細なドキュメントは[docs/README.md](docs/README.md)を参照してください。ドキュメントには以下の内容が含まれています：

- プロジェクト計画と設計ドキュメント
- アーキテクチャ設計（バックエンド、フロントエンド、インフラ）
- 開発ガイドラインとコーディング規約
- 運用とセキュリティポリシー

## 貢献ガイドライン

コントリビューションを歓迎します。詳細は[CONTRIBUTING.md](./CONTRIBUTING.md)をご覧ください。

## ライセンス

このプロジェクトは[MIT ライセンス](./LICENSE)の下で公開されています。

## セキュリティと個人情報保護

Vitascope は、医療データとライフメトリクスを扱うため、最高レベルのセキュリティ対策と個人情報保護措置を実施しています。詳細は[SECURITY.md](./SECURITY.md)をご覧ください。
