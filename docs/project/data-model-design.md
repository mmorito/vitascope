# Vitascope データモデル設計

このドキュメントでは、Vitascope アプリケーションのデータモデル設計と実装アプローチについて説明します。

## 実装アプローチ

Vitascope のデータモデルは段階的に実装します：

1. **フェーズ 1**: 内部データモデル（以下に定義）による実装
2. **フェーズ 2**: 内部データモデルと FHIR リソースのマッピング層実装
3. **フェーズ 3**: 完全な FHIR 準拠データモデルへの移行

詳細な FHIR データモデル設計については、[FHIR データモデル設計](./fhir-data-model.md)を参照してください。

## コアエンティティ（内部データモデル）

### User（ユーザー）

```json
{
  "id": "UUID",
  "username": "string",
  "email": "string",
  "full_name": "string",
  "date_of_birth": "date",
  "gender": "enum('male', 'female', 'other', 'prefer_not_to_say')",
  "height_cm": "number",
  "weight_kg": "number",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "settings": {
    "notifications_enabled": "boolean",
    "data_sharing_preference": "enum('none', 'anonymous', 'full')",
    "connected_devices": ["array of device ids"]
  }
}
```

### HealthData（健康データ）

```json
{
  "id": "UUID",
  "user_id": "UUID",
  "source": "string",
  "device_id": "string",
  "data_type": "enum('heart_rate', 'steps', 'sleep', 'blood_pressure', ...)",
  "timestamp": "timestamp",
  "value": "number",
  "unit": "string",
  "metadata": "json"
}
```

### Device（デバイス）

```json
{
  "id": "UUID",
  "user_id": "UUID",
  "name": "string",
  "type": "enum('smartwatch', 'fitness_tracker', 'smartphone', ...)",
  "manufacturer": "string",
  "model": "string",
  "last_synced_at": "timestamp",
  "status": "enum('active', 'inactive', 'disconnected')",
  "connection_details": "json"
}
```

### HealthInsight（健康インサイト）

```json
{
  "id": "UUID",
  "user_id": "UUID",
  "type": "enum('trend', 'anomaly', 'recommendation', ...)",
  "title": "string",
  "description": "string",
  "data_sources": ["array of data types"],
  "severity": "enum('info', 'low', 'medium', 'high')",
  "created_at": "timestamp",
  "read": "boolean",
  "related_data": "json"
}
```

## データベース設計

### リレーショナルスキーマ

Vitascope のデータは主に PostgreSQL で管理され、以下のテーブル構造を持ちます：

1. **users**: ユーザープロファイル情報
2. **devices**: 接続されたデバイス情報
3. **health_data**: 収集された健康データ（時系列データ）
4. **health_insights**: 分析された健康インサイト
5. **user_settings**: ユーザー設定と環境設定
6. **sharing_permissions**: データ共有権限設定

### インデックス戦略

効率的なクエリーパフォーマンスのために以下のインデックスを適用します：

1. **時系列データ用インデックス**

   - `(user_id, data_type, timestamp)` - 特定ユーザーのデータタイプ別時系列クエリ用
   - `(device_id, timestamp)` - デバイス別データ検索用

2. **分析クエリ用インデックス**
   - `(user_id, created_at)` - ユーザー別時系列インサイト用
   - `(data_type, value)` - データタイプ別の統計分析用

### JSONB フィールドの使用

PostgreSQL の JSONB 機能を活用して、柔軟なデータ構造をサポートします：

1. **メタデータ格納**

   - デバイス固有のデータ
   - プラットフォーム特有の追加情報
   - 将来的な拡張に対応する柔軟な構造

2. **JSONB 検索**
   - メタデータ内の特定フィールドでの検索をサポート
   - 複雑なデータ構造のクエリに対応

## データアクセス層

### リポジトリパターン

データアクセスはリポジトリパターンを使用して抽象化します：

```python
class HealthDataRepository:
    async def save(self, health_data: HealthData) -> HealthData:
        # データ保存ロジック

    async def find_by_user_and_type(self, user_id: UUID, data_type: str,
                                     start_date: datetime, end_date: datetime) -> List[HealthData]:
        # ユーザーと期間によるデータ検索ロジック

    async def get_latest_by_type(self, user_id: UUID, data_type: str) -> Optional[HealthData]:
        # 最新データ取得ロジック
```

### キャッシング戦略

頻繁にアクセスされるデータには Redis によるキャッシュ層を実装します：

1. **頻繁にアクセスされるデータのキャッシング**

   - ユーザープロファイル
   - 最新の健康メトリック
   - 集計済み統計データ

2. **キャッシュの無効化ポリシー**
   - 時間ベースの有効期限（TTL）
   - イベントベースの無効化（データ更新時）

## データバリデーションと整合性

1. **スキーマバリデーション**

   - Pydantic モデルによる入力データの検証
   - 型チェックと制約適用

2. **ビジネスルールの検証**

   - 生理学的に可能な値の範囲検証
   - タイムスタンプの論理的一貫性チェック

3. **参照整合性**
   - 外部キー制約によるデータの整合性保証
   - 孤立データの防止

## データマイグレーション戦略

1. **スキーマ変更の管理**

   - Alembic を使用したバージョン管理されたマイグレーション
   - ダウンタイムなしのデータベース変更

2. **レガシーデータの変換**
   - 過去データのスキーマアップグレード
   - データ変換のバックフィルプロセス
