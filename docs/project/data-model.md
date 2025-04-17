# Vitascope データモデル設計

このドキュメントでは、Vitascope アプリケーションで使用されるデータモデルの構造と設計について詳細に説明します。

## 概要

Vitascope のデータモデルは以下の主要カテゴリで構成されています：

1. **ユーザープロファイル** - ユーザー基本情報
2. **健康メトリック** - 健康関連の測定データ
3. **アクティビティデータ** - 運動・活動記録
4. **医療情報** - 診断、治療、薬剤情報
5. **外部連携データ** - 外部デバイス・API からのデータ
6. **分析・インサイト** - 分析結果とレコメンデーション

## エンティティ関係図 (ERD)

```
+---------------+       +---------------+       +---------------+
| ユーザー      |<----->| 健康メトリック |<----->| 外部データソース |
+---------------+       +---------------+       +---------------+
       |                       |                       |
       |                       |                       |
       v                       v                       v
+---------------+       +---------------+       +---------------+
| 医療情報      |<----->| アクティビティ  |<----->| インサイト    |
+---------------+       +---------------+       +---------------+
```

## 詳細データモデル

### 1. ユーザープロファイル (Users)

```json
{
  "id": "uuid",
  "auth_id": "string", // Cognito/認証プロバイダID
  "email": "string",
  "username": "string",
  "first_name": "string",
  "last_name": "string",
  "date_of_birth": "date",
  "gender": "enum", // male, female, other, prefer_not_to_say
  "height_cm": "float", // 身長（cm）
  "weight_kg": "float", // 体重（kg）
  "blood_type": "enum", // A, B, AB, O（+/-）
  "profile_picture_url": "string",
  "preferences": {
    "language": "string",
    "units": "enum", // metric, imperial
    "notification_settings": {
      "push_enabled": "boolean",
      "email_enabled": "boolean",
      "daily_summary": "boolean",
      "insight_alerts": "boolean"
    },
    "privacy_settings": {
      "data_sharing": "enum", // none, anonymous, all
      "third_party_access": "array" // 許可された第三者アプリID
    }
  },
  "medical_conditions": ["reference_to_medical_info"],
  "medications": ["reference_to_medications"],
  "emergency_contacts": [
    {
      "name": "string",
      "relationship": "string",
      "phone": "string",
      "email": "string"
    }
  ],
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "last_login": "timestamp"
}
```

### 2. 健康メトリック (HealthMetrics)

```json
{
  "id": "uuid",
  "user_id": "reference_to_user",
  "type": "enum", // heart_rate, blood_pressure, blood_glucose, etc.
  "timestamp": "timestamp", // 測定時刻
  "source": {
    "type": "enum", // manual, healthkit, google_fit, fitbit, etc.
    "device_id": "string",
    "app_id": "string"
  },
  "value": "float", // 測定値
  "unit": "string", // 単位（bpm, mmHg, mg/dL等）
  "metadata": {
    "measurement_method": "string",
    "position": "string", // 測定位置（該当する場合）
    "state": "string", // 状態（安静時、運動後等）
    "notes": "string"
  },
  // 血圧などの複合値の場合
  "components": {
    "systolic": "float",
    "diastolic": "float"
  },
  "tags": ["string"], // カスタムタグ
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

### 3. アクティビティデータ (Activities)

```json
{
  "id": "uuid",
  "user_id": "reference_to_user",
  "type": "enum", // walking, running, cycling, swimming, etc.
  "start_time": "timestamp",
  "end_time": "timestamp",
  "duration_seconds": "integer",
  "source": {
    "type": "enum", // manual, healthkit, google_fit, fitbit, etc.
    "device_id": "string",
    "app_id": "string"
  },
  "metrics": {
    "distance_meters": "float",
    "calories_burned": "float",
    "steps": "integer",
    "average_heart_rate": "float",
    "max_heart_rate": "float",
    "average_pace": "float", // 分/km
    "elevation_gain_meters": "float"
  },
  "location_data": {
    "start_location": {
      "latitude": "float",
      "longitude": "float"
    },
    "end_location": {
      "latitude": "float",
      "longitude": "float"
    },
    "path": ["array_of_lat_lng_objects"], // 詳細な経路（該当する場合）
    "elevation_profile": ["array_of_elevation_points"]
  },
  "zones": {
    "heart_rate_zones": [
      {
        "name": "string", // zone1, zone2, etc.
        "time_seconds": "integer"
      }
    ]
  },
  "weather": {
    "temperature_celsius": "float",
    "condition": "string",
    "humidity_percent": "float"
  },
  "notes": "string",
  "tags": ["string"],
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

### 4. 医療情報 (MedicalInfo)

```json
{
  "id": "uuid",
  "user_id": "reference_to_user",
  "type": "enum", // condition, allergy, surgery, vaccination, etc.

  // 病状・既往歴の場合
  "condition": {
    "name": "string",
    "status": "enum", // active, resolved, etc.
    "diagnosed_date": "date",
    "resolved_date": "date",
    "severity": "enum", // mild, moderate, severe
    "icd_code": "string" // 国際疾病分類コード
  },

  // アレルギーの場合
  "allergy": {
    "allergen": "string",
    "reaction": "string",
    "severity": "enum", // mild, moderate, severe
    "diagnosed_date": "date"
  },

  // 薬剤情報の場合
  "medication": {
    "name": "string",
    "generic_name": "string",
    "dosage": "string",
    "frequency": "string",
    "start_date": "date",
    "end_date": "date",
    "prescribed_by": "string",
    "reason": "string",
    "rxnorm_code": "string" // 薬剤コード
  },

  // 予防接種の場合
  "vaccination": {
    "name": "string",
    "date": "date",
    "manufacturer": "string",
    "lot_number": "string",
    "administered_by": "string"
  },

  "notes": "string",
  "attachments": [
    {
      "type": "string", // image, pdf, etc.
      "url": "string",
      "name": "string",
      "upload_date": "timestamp"
    }
  ],
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

### 5. 外部連携データ (ExternalSources)

```json
{
  "id": "uuid",
  "user_id": "reference_to_user",
  "type": "enum", // healthkit, google_fit, fitbit, hospital_api, etc.
  "source_id": "string", // 外部ソース固有ID
  "name": "string", // 表示名
  "connection_status": "enum", // connected, disconnected, error
  "last_sync": "timestamp",
  "auth_data": {
    "access_token": "encrypted_string",
    "refresh_token": "encrypted_string",
    "expires_at": "timestamp",
    "scope": "string"
  },
  "settings": {
    "sync_frequency": "enum", // realtime, hourly, daily
    "data_types": ["string"], // 同期するデータ種別
    "auto_sync": "boolean"
  },
  "metadata": {
    "device_model": "string",
    "os_version": "string",
    "app_version": "string"
  },
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

### 6. インサイトデータ (Insights)

```json
{
  "id": "uuid",
  "user_id": "reference_to_user",
  "type": "enum", // trend, alert, recommendation, achievement
  "category": "enum", // activity, heart, sleep, nutrition, etc.
  "title": "string",
  "description": "string",
  "severity": "enum", // info, warning, critical
  "generated_at": "timestamp",
  "expiry": "timestamp", // いつまで関連性があるか
  "data_sources": [
    {
      "type": "string", // health_metric, activity, medical, etc.
      "id": "uuid" // 参照元データID
    }
  ],
  "analysis": {
    "algorithm": "string", // 使用されたアルゴリズム
    "version": "string",
    "confidence": "float", // 0-1の信頼度
    "factors": [
      {
        "name": "string",
        "contribution": "float", // 寄与度
        "description": "string"
      }
    ],
    "visualization_data": {
      // グラフ描画用データ
    }
  },
  "actions": [
    {
      "type": "enum", // view_details, schedule_appointment, etc.
      "title": "string",
      "deep_link": "string"
    }
  ],
  "user_feedback": {
    "status": "enum", // none, helpful, not_helpful, dismissed
    "timestamp": "timestamp",
    "comment": "string"
  },
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

## FHIR マッピング

Vitascope のデータモデルは医療情報交換標準である FHIR (Fast Healthcare Interoperability Resources) と互換性を持たせるため、以下のマッピングを定義します。

### ユーザープロファイル → FHIR Patient

```json
{
  "resourceType": "Patient",
  "id": "[user.id]",
  "identifier": [
    {
      "system": "urn:vitascope:users",
      "value": "[user.id]"
    }
  ],
  "name": [
    {
      "family": "[user.last_name]",
      "given": ["[user.first_name]"]
    }
  ],
  "gender": "[user.gender]",
  "birthDate": "[user.date_of_birth]",
  "telecom": [
    {
      "system": "email",
      "value": "[user.email]"
    }
  ],
  "contact": [
    {
      "relationship": [
        {
          "text": "[user.emergency_contacts[0].relationship]"
        }
      ],
      "name": {
        "text": "[user.emergency_contacts[0].name]"
      },
      "telecom": [
        {
          "system": "phone",
          "value": "[user.emergency_contacts[0].phone]"
        }
      ]
    }
  ]
}
```

### 健康メトリック → FHIR Observation

```json
{
  "resourceType": "Observation",
  "id": "[health_metric.id]",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "vital-signs",
          "display": "Vital Signs"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "[loinc_code_for_metric_type]",
        "display": "[health_metric.type]"
      }
    ]
  },
  "subject": {
    "reference": "Patient/[health_metric.user_id]"
  },
  "effectiveDateTime": "[health_metric.timestamp]",
  "valueQuantity": {
    "value": "[health_metric.value]",
    "unit": "[health_metric.unit]",
    "system": "http://unitsofmeasure.org",
    "code": "[ucum_code_for_unit]"
  },
  "device": {
    "display": "[health_metric.source.device_id]"
  }
}
```

### アクティビティ → FHIR Observation

```json
{
  "resourceType": "Observation",
  "id": "[activity.id]",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "activity",
          "display": "Activity"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "[loinc_code_for_activity_type]",
        "display": "[activity.type]"
      }
    ]
  },
  "subject": {
    "reference": "Patient/[activity.user_id]"
  },
  "effectivePeriod": {
    "start": "[activity.start_time]",
    "end": "[activity.end_time]"
  },
  "component": [
    {
      "code": {
        "coding": [
          {
            "system": "http://loinc.org",
            "code": "41950-7",
            "display": "Physical activity (hours per day)"
          }
        ]
      },
      "valueQuantity": {
        "value": "[activity.duration_seconds/3600]",
        "unit": "h",
        "system": "http://unitsofmeasure.org",
        "code": "h"
      }
    },
    {
      "code": {
        "coding": [
          {
            "system": "http://loinc.org",
            "code": "41979-6",
            "display": "Calories burned"
          }
        ]
      },
      "valueQuantity": {
        "value": "[activity.metrics.calories_burned]",
        "unit": "kcal",
        "system": "http://unitsofmeasure.org",
        "code": "kcal"
      }
    }
  ]
}
```

### 医療情報 → FHIR Condition/AllergyIntolerance/MedicationStatement

医療情報のタイプに応じて適切な FHIR リソースにマッピングします。

#### 病状 → FHIR Condition

```json
{
  "resourceType": "Condition",
  "id": "[medical_info.id]",
  "clinicalStatus": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/condition-clinical",
        "code": "[medical_info.condition.status == 'active' ? 'active' : 'resolved']"
      }
    ]
  },
  "code": {
    "coding": [
      {
        "system": "http://hl7.org/fhir/sid/icd-10",
        "code": "[medical_info.condition.icd_code]",
        "display": "[medical_info.condition.name]"
      }
    ]
  },
  "subject": {
    "reference": "Patient/[medical_info.user_id]"
  },
  "onsetDateTime": "[medical_info.condition.diagnosed_date]",
  "abatementDateTime": "[medical_info.condition.resolved_date]"
}
```

## データアクセスパターン

Vitascope では、以下の主要なデータアクセスパターンを想定しています：

1. **ユーザープロファイル取得**

   - ユーザー ID による完全な個人プロファイル取得

2. **時系列メトリック取得**

   - ユーザー ID + メトリックタイプ + 期間による健康データ時系列取得
   - 例：過去 7 日間の心拍数データ

3. **アクティビティ履歴検索**

   - ユーザー ID + アクティビティタイプ + 期間によるフィルタリング
   - 例：先月の全てのランニングアクティビティ

4. **医療情報横断検索**

   - ユーザー ID による全ての医療情報（病状、薬剤等）の統合ビュー
   - 特定の医療条件に関連する健康メトリックとの相関関係

5. **インサイト生成・取得**
   - ユーザー ID に基づく関連インサイトの取得
   - 特定のメトリックに関連するインサイトのフィルタリング

## データ同期戦略

外部データソースとの同期は以下の戦略に基づいて実装します：

1. **定期的なポーリング同期**

   - HealthKit、Google Fit などからの定期的なデータポーリング
   - ユーザー設定に基づく同期頻度（リアルタイム、時間単位、日単位）

2. **プッシュベース同期**

   - Webhook やリアルタイム API が利用可能な場合のイベント駆動型同期
   - 変更通知に基づくリアルタイム更新

3. **増分同期**

   - 最終同期時刻以降の変更データのみを取得
   - ネットワークとストレージの効率化

4. **競合解決戦略**
   - タイムスタンプベースの競合解決
   - ソース信頼性に基づく優先順位付け
   - 手動解決が必要な場合のフラグ付け

## データストレージ実装

### プライマリストレージ

- **PostgreSQL** - トランザクション整合性が必要なコアデータ
  - JSONB カラムタイプを活用した柔軟なスキーマ
  - RLS（Row Level Security）によるマルチテナント分離

### セカンダリストレージ

- **Amazon S3** - 大容量バイナリデータ（画像、PDF 等）
- **Amazon TimeStream** - 時系列健康メトリックデータ
- **Amazon ElastiCache (Redis)** - キャッシュとリアルタイムデータ

## データセキュリティ

1. **保存データの暗号化**

   - 個人特定可能情報（PII）と PHI（Protected Health Information）の暗号化
   - ストレージレベルの暗号化（S3、RDS 等）

2. **転送中のデータ暗号化**

   - TLS/SSL による全ての通信の暗号化
   - API 通信における証明書ピンニング

3. **アクセス制御**

   - データの CRUD 操作に対する細粒度の IAM ポリシー
   - テナント分離と適切なアクセスコントロールリスト

4. **監査ログ**
   - PHI アクセスの包括的なログ記録
   - AWS CloudTrail による管理アクションの監視

## データバージョニングと履歴

1. **変更履歴追跡**

   - 重要な健康データおよび医療情報の変更履歴保持
   - 各レコードの変更者、変更日時、変更内容の記録

2. **ポイントインタイム復元**
   - データ破損や誤削除からの復旧機能
   - 特定時点のデータスナップショット参照

## バックアップ戦略

1. **定期的な完全バックアップ**

   - 日次の完全バックアップ
   - RDS の自動バックアップ

2. **継続的なトランザクションログ**
   - ポイントインタイム復元のための WAL ログ保持
   - クロスリージョンバックアップ

## データライフサイクル管理

1. **アーカイブポリシー**

   - 古いデータの低コストストレージへの移行
   - アクセス頻度に基づく階層化ストレージ

2. **保持ポリシー**
   - 規制要件に基づくデータ保持期間の設定
   - 期間満了後の適切なデータ削除
