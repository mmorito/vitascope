# Vitascope 外部データソース連携

このドキュメントでは、Vitascope アプリケーションと外部ヘルスケアデータソースとの連携について詳細に記述します。

## 概要

Vitascope は初期段階で以下の主要ヘルスケアプラットフォームとの連携を実装します：

1. Apple HealthKit (iOS)
2. Google Fit (Android)
3. Fitbit API

将来的な連携予定プラットフォーム：

- Garmin Connect
- Withings Health Mate
- Oura Ring

## 1. Apple HealthKit (iOS)

### 概要

Apple HealthKit は、iOS デバイス上で健康・フィットネスデータを管理する統合フレームワークです。

### データモデル構造

- **HKObjectType**: HealthKit で扱うデータタイプの基底クラス
  - **HKQuantityType**: 数値データ（歩数、心拍数、カロリーなど）
  - **HKCategoryType**: カテゴリデータ（睡眠分析、月経周期など）
  - **HKCorrelationType**: 複数の関連データ（血圧など）
  - **HKWorkoutType**: ワークアウトデータ
  - **HKDocumentType**: 医療文書データ

### 認証と権限

- アプリごとに具体的な読み書き権限が必要
- `NSHealthShareUsageDescription`と`NSHealthUpdateUsageDescription`を情報提供文（Info.plist）に記述
- `HKHealthStore`の`requestAuthorization`メソッドで具体的な項目へのアクセス権限をユーザーに要求

### データアクセス方法

```swift
// 権限リクエスト例
let healthStore = HKHealthStore()
let typesToShare: Set = [HKQuantityType.workoutType()]
let typesToRead: Set = [
    HKQuantityType.quantityType(forIdentifier: .heartRate)!,
    HKQuantityType.quantityType(forIdentifier: .stepCount)!
]

healthStore.requestAuthorization(toShare: typesToShare, read: typesToRead) { (success, error) in
    // ハンドリング
}

// データクエリ例
let queryType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
let query = HKStatisticsQuery(
    quantityType: queryType,
    quantitySamplePredicate: nil,
    options: .cumulativeSum
) { (_, result, _) in
    if let sum = result?.sumQuantity() {
        let steps = sum.doubleValue(for: HKUnit.count())
        // stepsを利用
    }
}
healthStore.execute(query)
```

### 制限事項

- iOS デバイスのみでの利用に限定
- バックグラウンドでの同期に制限あり（BackgroundDelivery を設定する必要）
- 一部データは読み取り専用（デバイス由来のデータなど）

### ベストプラクティス

- 過度な頻度でのデータポーリングを避ける
- バックグラウンド更新は`HKObserverQuery`と`enableBackgroundDelivery`を利用
- データ永続化前に重複チェックを行う
- ユーザーが HealthKit へのアクセスを拒否した場合の代替手段を提供する

## 2. Google Fit (Android)

### 概要

Google Fit は、Android デバイス向けの健康・フィットネスデータプラットフォームで、REST API と Android API を提供します。

### データモデル構造

- **DataTypes**: Fit データの基本単位
  - **DataSource**: データの送信元（アプリ、デバイスなど）
  - **DataPoint**: 特定の時間の測定値
  - **DataSet**: DataPoint の集合
  - **Session**: アクティビティセッション（ランニングなど）
  - **BleDevice**: Bluetooth Low Energy デバイス

### 認証と権限

- OAuth 2.0 認証を使用
- スコープごとの権限設定
  - `FITNESS_ACTIVITY_READ` - アクティビティデータ読み取り
  - `FITNESS_ACTIVITY_WRITE` - アクティビティデータ書き込み
  - `FITNESS_BODY_READ` - 体重などの身体データ読み取り
  - など

### データアクセス方法

```kotlin
// Fitness Optionsの構成
val fitnessOptions = FitnessOptions.builder()
    .addDataType(DataType.TYPE_STEP_COUNT_DELTA, FitnessOptions.ACCESS_READ)
    .addDataType(DataType.TYPE_HEART_RATE_BPM, FitnessOptions.ACCESS_READ)
    .build()

// 認証確認
if (!GoogleSignIn.hasPermissions(GoogleSignIn.getLastSignedInAccount(context), fitnessOptions)) {
    GoogleSignIn.requestPermissions(
        activity,
        REQUEST_CODE,
        GoogleSignIn.getLastSignedInAccount(context),
        fitnessOptions)
}

// データ読み取り例
val readRequest = DataReadRequest.Builder()
    .read(DataType.TYPE_STEP_COUNT_DELTA)
    .setTimeRange(startTime, endTime, TimeUnit.MILLISECONDS)
    .build()

Fitness.getHistoryClient(context, account)
    .readData(readRequest)
    .addOnSuccessListener { response ->
        // response.dataSetsからデータを処理
    }
```

### 制限事項

- 一部のデータアクセスは Highly Sensitive API と見なされ、Google 審査が必要
- バッチ操作に制限あり（最大 1000 操作/リクエスト）
- 短時間のデータポイントの記録には制限が存在

### ベストプラクティス

- 必要最小限のスコープのみ要求
- バッチ操作を活用して効率的にデータを処理
- データの重複を避けるために一意のセッション ID を使用
- バックグラウンド同期には WorkManager を利用

## 3. Fitbit API

### 概要

Fitbit API は、Fitbit デバイスからのデータにアクセスするための RESTful API を提供します。

### データモデル構造

Fitbit API では以下のリソースが提供されています：

- **Activities**: 歩数、距離、カロリー、アクティブ時間
- **Heart Rate**: 心拍数（安静時、運動時）
- **Sleep**: 睡眠ステージ、睡眠スコア
- **Body**: 体重、BMI、体脂肪率
- **Foods**: 食事記録、水分摂取量
- **Devices**: デバイス情報、バッテリーレベル

### 認証と権限

- OAuth 2.0 認証フロー
- スコープによる詳細な権限設定
  - `activity` - アクティビティデータ
  - `heartrate` - 心拍数データ
  - `sleep` - 睡眠データ
  - など

### データアクセス方法

```
// アクティビティデータの取得例
GET https://api.fitbit.com/1/user/-/activities/date/2023-01-01.json
Authorization: Bearer {access_token}

// 心拍数データの取得例
GET https://api.fitbit.com/1/user/-/activities/heart/date/2023-01-01/1d.json
Authorization: Bearer {access_token}
```

### 応答データ例（活動）

```json
{
  "activities": [],
  "goals": {
    "caloriesOut": 2500,
    "distance": 8.05,
    "floors": 10,
    "steps": 10000
  },
  "summary": {
    "activeScore": -1,
    "activityCalories": 1306,
    "caloriesBMR": 1572,
    "caloriesOut": 2878,
    "distances": [
      { "activity": "total", "distance": 8.43 },
      { "activity": "tracker", "distance": 8.43 },
      { "activity": "loggedActivities", "distance": 0 },
      { "activity": "veryActive", "distance": 3.52 },
      { "activity": "moderatelyActive", "distance": 2.37 },
      { "activity": "lightlyActive", "distance": 2.54 },
      { "activity": "sedentaryActive", "distance": 0 }
    ],
    "elevation": 21.34,
    "fairlyActiveMinutes": 31,
    "floors": 7,
    "heartRateZones": [
      {
        "caloriesOut": 1138.6,
        "max": 94,
        "min": 30,
        "minutes": 797,
        "name": "標準以下"
      },
      {
        "caloriesOut": 498.4,
        "max": 132,
        "min": 94,
        "minutes": 40,
        "name": "脂肪燃焼"
      },
      {
        "caloriesOut": 223.6,
        "max": 160,
        "min": 132,
        "minutes": 11,
        "name": "有酸素運動"
      },
      {
        "caloriesOut": 90.1,
        "max": 220,
        "min": 160,
        "minutes": 2,
        "name": "最大心拍数"
      }
    ],
    "lightlyActiveMinutes": 213,
    "marginalCalories": 807,
    "restingHeartRate": 68,
    "sedentaryMinutes": 588,
    "steps": 11218,
    "veryActiveMinutes": 29
  }
}
```

### 制限事項

- API リクエスト制限: 150 リクエスト/時間
- 一部データへのアクセスには Fitbit 開発者規約の同意が必要
- 過去データへのアクセスは 150 日間に制限される場合あり

### ベストプラクティス

- レート制限を考慮した効率的なデータ取得
- サブスクリプション API を使用してリアルタイム更新を受信
- メタデータを活用してデータ同期の最適化
- トークンの安全な保存と更新メカニズムの実装

## データ統合アプローチ

各プラットフォームから取得したデータを統合するために以下のアプローチを採用します：

### 1. 標準化レイヤー

- 各プラットフォーム固有のデータ形式を Vitascope 内部モデルに変換
- 単位やフォーマットの標準化（例：歩数、心拍数、睡眠段階）

### 2. 重複排除メカニズム

- タイムスタンプとデータソースを利用した重複データの特定と排除
- 競合するデータの場合は、より高精度なデバイスを優先

### 3. メタデータ管理

- データソース、精度、信頼性などのメタデータを保存
- 将来的な FHIR マッピングに必要な情報の保持

### 4. 段階的同期

- 初回同期: 過去データを一括取得（利用可能な範囲内）
- 定期同期: スケジュールに基づく定期的なデータ取得
- リアルタイム同期: プッシュ通知またはポーリングによる迅速な更新
