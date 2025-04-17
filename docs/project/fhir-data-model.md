# Vitascope FHIR データモデル設計

## 概要

このドキュメントでは、Vitascope アプリケーションのデータモデルを、医療データ交換の世界標準規格である HL7 FHIR（Fast Healthcare Interoperability Resources）に準拠して設計するためのガイドラインを提供します。このアプローチにより、将来的に外部の医療システムとのデータ連携が容易になります。

## FHIR の基本概念

FHIR は以下の主要コンポーネントで構成されています：

1. **リソース**: 患者情報や測定値などの医療データを表現する標準化された構造体
2. **データ型**: 文字列、整数、コード化された値などの基本データ型
3. **拡張機能**: 標準リソースを拡張するためのメカニズム
4. **プロファイル**: 特定のユースケースのためにリソースをカスタマイズする仕組み

## Vitascope で利用する FHIR リソース

### 1. 患者情報管理（Patient）

```json
{
  "resourceType": "Patient",
  "id": "patient-001",
  "identifier": [
    {
      "system": "urn:oid:1.2.392.200119.6.102.11234567890",
      "value": "12345"
    }
  ],
  "active": true,
  "name": [
    {
      "family": "山田",
      "given": ["太郎"]
    }
  ],
  "gender": "male",
  "birthDate": "1970-01-01",
  "telecom": [
    {
      "system": "phone",
      "value": "090-1234-5678"
    },
    {
      "system": "email",
      "value": "yamada@example.org"
    }
  ],
  "extension": [
    {
      "url": "http://vitascope.org/fhir/StructureDefinition/user-settings",
      "valueString": "{\"notifications_enabled\": true, \"data_sharing_preference\": \"anonymous\"}"
    }
  ]
}
```

#### マッピング（既存データモデル → FHIR）

| Vitascope モデル | FHIR パス                                                                            |
| ---------------- | ------------------------------------------------------------------------------------ |
| id               | Patient.id                                                                           |
| username         | Patient.identifier[0].value                                                          |
| email            | Patient.telecom[system='email'].value                                                |
| full_name        | Patient.name[0].family + Patient.name[0].given[0]                                    |
| date_of_birth    | Patient.birthDate                                                                    |
| gender           | Patient.gender                                                                       |
| settings         | Patient.extension[url='http://vitascope.org/fhir/StructureDefinition/user-settings'] |

### 2. 健康データ管理（Observation）

```json
{
  "resourceType": "Observation",
  "id": "heart-rate-001",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "vital-signs",
          "display": "バイタルサイン"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "8867-4",
        "display": "心拍数"
      }
    ]
  },
  "subject": {
    "reference": "Patient/patient-001"
  },
  "effectiveDateTime": "2023-06-01T12:30:00+09:00",
  "valueQuantity": {
    "value": 72,
    "unit": "beats/minute",
    "system": "http://unitsofmeasure.org",
    "code": "/min"
  },
  "device": {
    "reference": "Device/smartwatch-001"
  }
}
```

#### マッピング（既存データモデル → FHIR）

| Vitascope モデル | FHIR パス                       |
| ---------------- | ------------------------------- |
| id               | Observation.id                  |
| user_id          | Observation.subject.reference   |
| source           | Observation.device.reference    |
| device_id        | Observation.device.reference    |
| data_type        | Observation.code.coding[0].code |
| timestamp        | Observation.effectiveDateTime   |
| value            | Observation.valueQuantity.value |
| unit             | Observation.valueQuantity.unit  |
| metadata         | 拡張として実装                  |

### 3. デバイス情報管理（Device）

```json
{
  "resourceType": "Device",
  "id": "smartwatch-001",
  "identifier": [
    {
      "system": "http://vitascope.org/fhir/device-identifiers",
      "value": "APPLE-WATCH-SN12345"
    }
  ],
  "status": "active",
  "manufacturer": "Apple",
  "modelNumber": "Watch Series 7",
  "type": {
    "coding": [
      {
        "system": "http://vitascope.org/fhir/device-types",
        "code": "smartwatch",
        "display": "スマートウォッチ"
      }
    ]
  },
  "owner": {
    "reference": "Patient/patient-001"
  },
  "lastSystemChange": "2023-06-01T12:00:00+09:00"
}
```

#### マッピング（既存データモデル → FHIR）

| Vitascope モデル   | FHIR パス                  |
| ------------------ | -------------------------- |
| id                 | Device.id                  |
| user_id            | Device.owner.reference     |
| name               | Device.identifier[0].value |
| type               | Device.type.coding[0].code |
| manufacturer       | Device.manufacturer        |
| model              | Device.modelNumber         |
| last_synced_at     | Device.lastSystemChange    |
| status             | Device.status              |
| connection_details | 拡張として実装             |

### 4. 健康インサイト管理（DetectedIssue/GuidanceResponse）

```json
{
  "resourceType": "DetectedIssue",
  "id": "insight-001",
  "status": "final",
  "code": {
    "coding": [
      {
        "system": "http://vitascope.org/fhir/insight-types",
        "code": "trend",
        "display": "健康トレンド"
      }
    ]
  },
  "severity": "moderate",
  "patient": {
    "reference": "Patient/patient-001"
  },
  "identifiedDateTime": "2023-06-02T10:00:00+09:00",
  "evidence": [
    {
      "detail": [
        {
          "reference": "Observation/heart-rate-001"
        }
      ]
    }
  ],
  "detail": "過去1週間の心拍数が平均より10%上昇しています。"
}
```

#### マッピング（既存データモデル → FHIR）

| Vitascope モデル | FHIR パス                                 |
| ---------------- | ----------------------------------------- |
| id               | DetectedIssue.id                          |
| user_id          | DetectedIssue.patient.reference           |
| type             | DetectedIssue.code.coding[0].code         |
| title            | DetectedIssue.code.coding[0].display      |
| description      | DetectedIssue.detail                      |
| data_sources     | DetectedIssue.evidence.detail[].reference |
| severity         | DetectedIssue.severity                    |
| created_at       | DetectedIssue.identifiedDateTime          |
| read             | 拡張として実装                            |
| related_data     | 拡張または追加のリソース参照として実装    |

### 5. 活動記録管理（Observation with activity category）

```json
{
  "resourceType": "Observation",
  "id": "activity-001",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "activity",
          "display": "活動"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "41950-7",
        "display": "歩数"
      }
    ]
  },
  "subject": {
    "reference": "Patient/patient-001"
  },
  "effectiveDateTime": "2023-06-01T00:00:00+09:00",
  "valueQuantity": {
    "value": 8500,
    "unit": "steps",
    "system": "http://unitsofmeasure.org",
    "code": "steps"
  }
}
```

### 6. 睡眠データ管理（Observation with sleep components）

```json
{
  "resourceType": "Observation",
  "id": "sleep-001",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "sleep",
          "display": "睡眠"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "93832-4",
        "display": "睡眠時間"
      }
    ]
  },
  "subject": {
    "reference": "Patient/patient-001"
  },
  "effectiveDateTime": "2023-06-01T00:00:00+09:00",
  "component": [
    {
      "code": {
        "coding": [
          {
            "system": "http://vitascope.org/fhir/sleep-metrics",
            "code": "deep-sleep",
            "display": "深い睡眠"
          }
        ]
      },
      "valueQuantity": {
        "value": 120,
        "unit": "min",
        "system": "http://unitsofmeasure.org",
        "code": "min"
      }
    },
    {
      "code": {
        "coding": [
          {
            "system": "http://vitascope.org/fhir/sleep-metrics",
            "code": "rem-sleep",
            "display": "レム睡眠"
          }
        ]
      },
      "valueQuantity": {
        "value": 90,
        "unit": "min",
        "system": "http://unitsofmeasure.org",
        "code": "min"
      }
    }
  ]
}
```

## 標準コード体系の利用

Vitascope では以下の標準コード体系を活用します：

1. **LOINC**: 検査・観察項目の標準コード

   - 心拍数: 8867-4
   - 歩数: 41950-7
   - 睡眠時間: 93832-4
   - 体重: 29463-7
   - 血圧収縮期: 8480-6
   - 血圧拡張期: 8462-4
   - 血中酸素飽和度: 59408-5

2. **SNOMED CT**: 医療用語の標準コード

   - 各種疾患や状態の記述に利用

3. **UCUM**: 単位の標準コード
   - /min: 毎分（心拍数など）
   - steps: 歩数
   - kg: キログラム（体重）
   - min: 分（時間）

## 実装アプローチ

### 1. 段階的アプローチ

1. **フェーズ 1**: 内部データモデルの確立

   - アプリケーションのニーズを満たす独自データモデルを実装
   - FHIR 互換性を考慮した設計

2. **フェーズ 2**: マッピングレイヤーの実装

   - 内部データモデルと FHIR リソース間の変換機能
   - 外部 API の FHIR フォーマット対応

3. **フェーズ 3**: 完全な FHIR 対応
   - データモデルの FHIR リソースへの移行
   - 全 API エンドポイントの FHIR 対応

### 2. 日本固有の対応

1. **JP Core プロファイルの活用**

   - 日本医療情報学会 FHIR 日本実装検討 WG の定義に準拠
   - 日本の医療制度に特化した拡張を採用

2. **日本語対応**
   - マルチバイト文字のサポート
   - 日本語表示名と説明の追加

### 3. セキュリティ考慮事項

1. **認証・認可**

   - OAuth2.0/OpenID Connect の採用
   - スコープベースのアクセス制御

2. **プライバシー保護**
   - 個人情報保護法への準拠
   - 同意管理の実装

## API エンドポイント設計

FHIR に準拠した RESTful API エンドポイントを以下のように設計します：

```
GET /fhir/Patient/:id - 患者情報の取得
GET /fhir/Observation?subject=Patient/:id - 患者の観測データ取得
GET /fhir/Device?owner=Patient/:id - 患者のデバイス情報取得
GET /fhir/DetectedIssue?patient=Patient/:id - 患者の健康インサイト取得
```

## まとめ

HL7 FHIR リソースをベースにした Vitascope のデータモデル設計により、以下のメリットが得られます：

1. **相互運用性**: 医療情報システムとの容易なデータ交換
2. **標準化**: 国際標準に基づく一貫したデータ構造
3. **拡張性**: 将来のニーズに応じた柔軟な拡張
4. **データ統合**: 異なるソースからのヘルスケアデータの統合

FHIR は複雑な規格ですが、このドキュメントで概説した段階的アプローチにより、Vitascope の主要機能に焦点を当てながら、標準準拠のデータモデルを構築することができます。
