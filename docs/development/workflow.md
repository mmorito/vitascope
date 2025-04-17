# 開発ワークフロー

効率的で一貫性のある開発プロセスを確保するため、以下の開発ワークフローに従ってください。

## 環境設定

### 開発環境セットアップ

1. 必須ソフトウェアのインストール：
   - Node.js LTSバージョン（フロントエンド）
   - Python 3.9+（バックエンド）
   - Poetry（Pythonパッケージ管理）
   - Docker と Docker Compose
   - AWS CLI
   - LocalStack（AWSサービスのローカルエミュレーション）

2. リポジトリのクローン：
   ```bash
   git clone https://github.com/company/vitascope.git
   cd vitascope
   ```

3. 初期設定スクリプトの実行：
   ```bash
   ./setup.sh
   ```

4. 依存関係のインストール：
   ```bash
   # フロントエンド
   cd frontend
   npm install
   
   # バックエンド
   cd backend
   poetry install
   ```

5. 環境変数の設定：
   各ディレクトリの`.env.example`を`.env`にコピーして必要な値を設定

6. 開発サーバーの起動：
   ```bash
   # 全環境（Docker Compose）
   docker-compose up
   
   # フロントエンドのみ
   cd frontend
   npm start
   
   # バックエンドのみ
   cd backend
   poetry run uvicorn app.main:app --reload
   ```

## ブランチ戦略

以下のブランチ戦略を採用しています：

- **main**: 本番環境用ブランチ。常にデプロイ可能な状態を維持。
- **develop**: 開発用ブランチ。CI/CDによる自動テストが通過したコードのみをマージ。
- **feature/[機能名]**: 新機能開発用。例：`feature/user-authentication`
- **bugfix/[バグID]**: バグ修正用。例：`bugfix/login-error-123`
- **release/[バージョン]**: リリース準備用。例：`release/v1.0.0`
- **hotfix/[問題ID]**: 本番環境の緊急修正用。例：`hotfix/critical-security-456`

## 開発プロセス

### 1. 課題の割り当て

1. GitHubのIssueを確認して作業するタスクを選択
2. 自分にIssueをアサイン
3. タスクを「進行中」ステータスに変更

### 2. 開発作業

1. 最新の`develop`ブランチからフィーチャーブランチを作成：
   ```bash
   git checkout develop
   git pull
   git checkout -b feature/my-feature
   ```

2. ローカル環境での開発：
   - テスト駆動開発（TDD）の原則に従う
   - コードレビュー基準を満たすコードを書く
   - 適切なテストを作成

3. 変更内容のコミット：
   ```bash
   git add .
   git commit -m "feat: add user authentication"  # Conventional Commitsに従う
   ```

4. 定期的に変更をリモートにプッシュ：
   ```bash
   git push origin feature/my-feature
   ```

### 3. コードレビュー

1. GitHubでプルリクエスト（PR）を作成：
   - ベースブランチは `develop`
   - PRタイトルと説明は明確に記述
   - 関連するIssueへのリンクを含める

2. 自動テストとCI/CDが実行されるのを待つ

3. レビュアーを指定してレビューを依頼

4. レビューコメントに基づいて修正

5. 承認後、PRをマージ（スカッシュマージを推奨）

### 4. テストとデプロイ

1. `develop`ブランチへのマージ後、ステージング環境に自動デプロイ

2. テスト環境でQAテストを実施

3. リリース時：
   ```bash
   git checkout develop
   git pull
   git checkout -b release/v1.0.0
   # バージョン番号の更新など必要な変更
   git push origin release/v1.0.0
   ```

4. リリースブランチのPRを`main`ブランチに対して作成

5. 承認と最終テスト後、`main`へマージすると本番環境にデプロイ

### 5. リリース後

1. 本番環境での検証

2. `main`の変更を`develop`にマージして同期

3. 完了したIssueをクローズ

4. 次のスプリントの計画に参加

## コミットメッセージ規約

Conventional Commitsの形式に従います：

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### タイプ

- **feat**: 新機能
- **fix**: バグ修正
- **docs**: ドキュメント変更
- **style**: コードスタイル変更（フォーマットなど）
- **refactor**: リファクタリング（機能追加や修正なし）
- **test**: テスト関連
- **chore**: ビルドプロセスや補助ツール変更
- **perf**: パフォーマンス改善
- **ci**: CI設定変更
- **build**: ビルドシステム変更

例：
```
feat(auth): add user registration flow

Implement the user registration workflow with email verification.
This includes:
- Registration form
- Email verification
- Welcome email

Fixes #123
```

## ドキュメント作成

すべての主要機能には以下のドキュメントを作成します：

1. **技術設計書**：アーキテクチャと実装詳細
2. **API仕様書**：エンドポイント、リクエスト/レスポンス形式
3. **ユーザーガイド**：機能の使用方法
4. **データフロー図**：データの流れと処理

## レビュープロセス

### コードレビュー基準

1. 機能要件を満たしているか
2. コード規約に準拠しているか
3. セキュリティ上の問題がないか
4. パフォーマンスに問題がないか
5. 適切なテストが書かれているか
6. エラーハンドリングが適切か
7. ログ出力が適切か
8. ドキュメントが更新されているか

### レビューコメントのエチケット

- 具体的かつ建設的なフィードバックを提供
- コードだけでなく、アイデアを評価
- 質問形式を活用（「これを〇〇に変更した方が良いのでは？」）
- ポジティブな点も指摘
- 個人ではなくコードに焦点を当てる 