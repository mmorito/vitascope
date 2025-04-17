# 監視とアラート戦略

Vitascope システムの安定性と性能を確保するため、包括的な監視・アラート体制を構築します。

## 監視戦略

### 監視レイヤー

1. **インフラストラクチャ監視**

   - ハードウェアリソース（CPU、メモリ、ディスク、ネットワーク）
   - クラウドサービス状態
   - コンテナヘルス

2. **アプリケーション監視**

   - API 応答時間
   - エラーレート
   - スループット
   - リクエスト数

3. **ビジネスメトリクス**

   - アクティブユーザー数
   - 機能ごとの使用状況
   - データ同期成功率
   - 重要業務プロセスの完了率

4. **セキュリティ監視**
   - 認証失敗
   - 異常なアクセスパターン
   - リソースへの不正アクセス試行
   - 設定変更

### 監視指標

#### インフラ監視指標

| メトリクス               | 説明                       | 閾値             | アラートレベル  |
| ------------------------ | -------------------------- | ---------------- | --------------- | ---- |
| CPU 使用率               | インスタンスの CPU 使用率  | 70%以上          | 警告<br>90%以上 | 緊急 |
| メモリ使用率             | インスタンスのメモリ使用率 | 80%以上          | 警告<br>95%以上 | 緊急 |
| ディスク使用率           | ストレージの使用率         | 80%以上          | 警告<br>90%以上 | 緊急 |
| ネットワークトラフィック | 送受信トラフィック量       | 通常の 3 倍以上  | 警告            |
| ECS サービス状態         | Fargate サービスの状態     | unhealthy        | 緊急            |
| RDS 接続数               | データベース接続数         | 最大接続数の 80% | 警告            |
| SQS キュー深度           | キュー内のメッセージ数     | 1000 以上        | 警告            |
| Lambda 実行エラー        | Lambda 関数の実行エラー率  | 1%以上           | 警告<br>5%以上  | 緊急 |

#### アプリケーション監視指標

| メトリクス           | 説明                             | 閾値                      | アラートレベル  |
| -------------------- | -------------------------------- | ------------------------- | --------------- | ---- |
| API 応答時間         | API 呼び出しの応答時間の P95     | 500ms 以上                | 警告<br>1s 以上 | 緊急 |
| エラーレート         | 全リクエストに占めるエラーの割合 | 1%以上                    | 警告<br>5%以上  | 緊急 |
| ユーザーセッション数 | アクティブセッション数           | 前日比 30%以上減少        | 警告            |
| 認証失敗             | 認証失敗の回数                   | 同一 IP から連続 5 回以上 | 警告            |
| API 呼び出し頻度     | 単位時間あたりの API 呼び出し数  | 通常の 2 倍以上           | 警告            |
| 依存サービス可用性   | 外部依存サービスの状態           | 障害検出                  | 緊急            |

#### ビジネスメトリクス

| メトリクス             | 説明                              | 閾値               | アラートレベル |
| ---------------------- | --------------------------------- | ------------------ | -------------- |
| 日次アクティブユーザー | 24 時間以内のアクティブユーザー数 | 前週比 20%以上減少 | 警告           |
| データ同期失敗率       | 健康データ同期の失敗率            | 5%以上             | 警告           |
| 新規登録数             | 新規ユーザー登録数                | 前週比 30%以上減少 | 情報           |
| 機能使用率             | 主要機能の使用率                  | 前週比 40%以上減少 | 情報           |

## 監視ツール

### 主要ツール

- **AWS CloudWatch**: インフラと AWS サービスの監視

  - メトリクス収集
  - ログ集約
  - アラート設定
  - ダッシュボード作成

- **Prometheus + Grafana**: アプリケーション監視

  - カスタムメトリクス収集
  - リアルタイム可視化
  - 高度なダッシュボード
  - アラート管理

- **AWS X-Ray**: 分散トレーシング

  - マイクロサービス間の依存関係可視化
  - パフォーマンスボトルネック特定
  - エラー根本原因分析

- **ELK Stack**: ログ解析
  - ログ集約と検索
  - パターン認識
  - 異常検出

### 監視ツール設定例

#### Prometheus & Grafana 設定

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "vitascope-api"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["api:8000"]
        labels:
          service: "api"
          environment: "production"

  - job_name: "vitascope-worker"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["worker:8080"]
        labels:
          service: "worker"
          environment: "production"

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

#### CloudWatch アラーム設定（Terraform）

```hcl
# cloudwatch_alarms.tf
resource "aws_cloudwatch_metric_alarm" "api_high_cpu" {
  alarm_name          = "vitascope-api-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "60"
  statistic           = "Average"
  threshold           = "70"
  alarm_description   = "API CPU使用率が70%を超えています"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.api.name
  }
}

resource "aws_cloudwatch_metric_alarm" "api_response_time" {
  alarm_name          = "vitascope-api-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "3"
  metric_name         = "p95ResponseTime"
  namespace           = "Vitascope/API"
  period              = "60"
  statistic           = "Average"
  threshold           = "500"
  alarm_description   = "API応答時間が500msを超えています"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
}
```

#### FastAPI アプリケーションでの Prometheus 統合

```python
# metrics.py
from prometheus_client import Counter, Histogram, start_http_server
import time

# リクエストカウンタ
REQUEST_COUNT = Counter(
    'app_request_count',
    'アプリケーションリクエスト数',
    ['method', 'endpoint', 'status_code']
)

# レスポンスタイムヒストグラム
REQUEST_TIME = Histogram(
    'app_request_latency_seconds',
    'アプリケーションリクエスト処理時間',
    ['method', 'endpoint']
)

# FastAPIミドルウェア
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    # リクエスト時間計測開始
    start_time = time.time()

    # リクエスト処理
    response = await call_next(request)

    # メトリクス記録
    request_time = time.time() - start_time
    REQUEST_TIME.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(request_time)

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status_code=response.status_code
    ).inc()

    return response

# メトリクスエンドポイント
@app.get("/metrics")
async def metrics():
    from prometheus_client import generate_latest
    return Response(
        content=generate_latest(),
        media_type="text/plain"
    )
```

#### Grafana ダッシュボード設定（JSON）

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "デプロイ",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "panels": [
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 2,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.2.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(app_request_latency_seconds_bucket{endpoint=~\"/api/v1/.*\"}[5m])) by (le, endpoint))",
          "interval": "",
          "legendFormat": "{{endpoint}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "API応答時間 (P95)",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "schemaVersion": 26,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Vitascope API Performance",
  "uid": "vitascope-api",
  "version": 1
}
```

### CI/CD パイプラインとの連携

監視システムと CI/CD パイプラインを統合し、デプロイと監視を連携させます：

```yaml
# GitHub Actions ワークフロー例
name: Deploy and Monitor

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      # ビルド・テスト・デプロイステップ...

      # デプロイ時のアノテーション作成
      - name: Create Deployment Annotation
        run: |
          VERSION=$(echo $GITHUB_SHA | cut -c1-7)
          DEPLOY_TIME=$(date +%s000)

          curl -X POST \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ secrets.GRAFANA_API_KEY }}" \
            -d '{
              "dashboardId": 1,
              "time": '$DEPLOY_TIME',
              "tags": ["deployment", "github-actions"],
              "text": "Deployed version '$VERSION'"
            }' \
            https://grafana.example.com/api/annotations

      # デプロイ後の健全性チェック
      - name: Post-deployment Health Check
        run: |
          # 基本的なエンドポイント健全性チェック
          curl -f https://api.vitascope.com/health

          # Prometheusメトリクスのチェック (5分間の監視)
          for i in {1..5}; do
            # エラー率のチェック
            ERROR_RATE=$(curl -s https://prometheus.vitascope.com/api/v1/query?query=sum(rate(app_request_count{status_code=~"5.."}[1m]))/sum(rate(app_request_count[1m]))*100 | jq '.data.result[0].value[1]')
            
            if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
              echo "Error rate too high after deployment: $ERROR_RATE%"
              # 自動ロールバックトリガー
              exit 1
            fi
            
            echo "Health check $i - Error rate: $ERROR_RATE%"
            sleep 60
          done
```

### 実装方針

1. **Infrastructure コードによる監視設定**

   - Terraform によるモニタリングリソース管理
   - GitOps によるアラート設定のバージョン管理

2. **カスタムメトリクス**

   - アプリケーションからの重要メトリクスの出力
   - ビジネスプロセスごとの成功/失敗指標の収集

3. **サービスヘルスチェック**

   - 主要エンドポイントの定期的なヘルスチェック
   - 合成モニタリングによるユーザーフロー検証

4. **可観測性の 3 本柱**
   - メトリクス: 数値データでシステム状態を把握
   - ログ: 詳細な動作記録を分析
   - トレース: 分散システム間の処理フローを追跡

## アラート体制

### 緊急度レベル

| レベル | 名称 | 説明                                                 | 応答時間        | 通知方法                 |
| ------ | ---- | ---------------------------------------------------- | --------------- | ------------------------ |
| P1     | 緊急 | サービス停止、データ漏洩など<br>即時対応が必要な問題 | 即時（24 時間） | 電話 + メール + チャット |
| P2     | 高   | 特定機能の障害、性能劣化<br>業務影響が大きい問題     | 4 時間以内      | メール + チャット        |
| P3     | 中   | 非クリティカルな問題<br>一部機能に影響               | 24 時間以内     | メール + チャット        |
| P4     | 低   | 軽微な問題<br>即時対応不要                           | 次のスプリント  | チケット                 |

### オンコール体制

- **ローテーション**: 週単位のオンコール当番制
- **エスカレーションパス**: 一次対応 → 二次対応 → 管理者
- **対応時間**: 24 時間 365 日（P1・P2 のみ）
- **引き継ぎプロセス**: 当番交代時の明確な引き継ぎ手順
- **オンコールツール**: PagerDuty

### PagerDuty 設定例

```yaml
# PagerDuty Escalation Policy
- name: "Vitascope Production Escalation"
  description: "本番環境の障害対応エスカレーションポリシー"
  escalation_rules:
    - escalation_delay_in_minutes: 30
      targets:
        - type: "user_reference"
          id: "PRIMARY_ON_CALL_USER_ID"
    - escalation_delay_in_minutes: 30
      targets:
        - type: "user_reference"
          id: "SECONDARY_ON_CALL_USER_ID"
    - escalation_delay_in_minutes: 30
      targets:
        - type: "user_reference"
          id: "ENGINEERING_MANAGER_ID"

# PagerDuty Service Configuration
- name: "Vitascope API Service"
  description: "Vitascope APIサービス監視"
  escalation_policy_id: "ESCALATION_POLICY_ID"
  alert_creation: "create_alerts_and_incidents"
  incident_urgency_rule:
    type: "constant"
    urgency: "high"
  alert_grouping_parameters:
    type: "intelligent"
  integrations:
    - type: "cloudwatch_inbound_integration"
      name: "CloudWatch Integration"
    - type: "prometheus_inbound_integration"
      name: "Prometheus Integration"
```

### インシデント対応

1. **検知**:

   - アラート通知の受信
   - 状況の初期評価

2. **トリアージ**:

   - 影響範囲の特定
   - 緊急度の判断
   - 必要なチームのエスカレーション

3. **調査**:

   - ログ・メトリクスの分析
   - 根本原因の特定
   - 一時的な対応策の検討

4. **解決**:

   - 対応策の実施
   - 影響の軽減
   - サービス復旧

5. **事後分析**:
   - インシデントレポートの作成
   - 再発防止策の検討
   - モニタリング改善の提案

### インシデント対応の自動化

重大なアラートに対する初期対応を自動化するスクリプト例：

```python
# incident_responder.py
import boto3
import requests
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """CloudWatchアラームから起動されるLambda関数"""

    alarm_name = event['detail']['alarmName']
    alarm_description = event['detail'].get('alarmDescription', '')
    alarm_reason = event['detail'].get('newStateReason', '')

    logger.info(f"Incident response triggered for alarm: {alarm_name}")

    # インシデントの種類を特定
    if 'api-high-cpu' in alarm_name:
        # API高負荷インシデント
        autoscale_response = trigger_autoscaling()
        create_incident_ticket(
            title=f"高CPU使用率アラート: {alarm_name}",
            description=f"{alarm_description}\n\n{alarm_reason}",
            priority="P2",
            auto_action=f"Auto-scaling triggered: {autoscale_response}"
        )

    elif 'database-connection' in alarm_name:
        # データベース接続インシデント
        connection_reset_response = reset_database_connections()
        create_incident_ticket(
            title=f"データベース接続アラート: {alarm_name}",
            description=f"{alarm_description}\n\n{alarm_reason}",
            priority="P1",
            auto_action=f"DB connections reset: {connection_reset_response}"
        )

    else:
        # その他のインシデント
        create_incident_ticket(
            title=f"アラート: {alarm_name}",
            description=f"{alarm_description}\n\n{alarm_reason}",
            priority="P3"
        )

    return {
        'statusCode': 200,
        'body': json.dumps('Incident response initiated')
    }

def trigger_autoscaling():
    """Auto Scalingの容量を一時的に増加"""
    client = boto3.client('application-autoscaling')
    response = client.register_scalable_target(
        ServiceNamespace='ecs',
        ResourceId='service/vitascope-cluster/vitascope-api',
        ScalableDimension='ecs:service:DesiredCount',
        MinCapacity=4,  # 一時的に最小容量を増加
        MaxCapacity=8   # 一時的に最大容量を増加
    )
    return "Capacity temporarily increased"

def reset_database_connections():
    """問題のあるデータベース接続をリセット"""
    rds = boto3.client('rds')
    response = rds.reboot_db_instance(
        DBInstanceIdentifier='vitascope-db'
    )
    return "Database connections reset initiated"

def create_incident_ticket(title, description, priority, auto_action=None):
    """インシデントチケットを作成"""
    # JIRAやPagerDutyなどのチケットシステムAPIを呼び出す
    ticket_data = {
        'title': title,
        'description': description,
        'priority': priority,
        'auto_action': auto_action,
        'status': 'open'
    }

    # チケットシステムへのAPI呼び出し
    # response = requests.post('https://ticketing-system.example.com/api/incidents',
    #                        json=ticket_data,
    #                        headers={'Authorization': 'Bearer ' + API_TOKEN})

    logger.info(f"Incident ticket created: {title} with priority {priority}")
    if auto_action:
        logger.info(f"Automatic action taken: {auto_action}")

    return "Ticket created"
```

## ダッシュボード

### 運用ダッシュボード

- **サービス概況**: 主要サービスのヘルスステータス
- **リソース使用状況**: CPU、メモリ、ディスク、ネットワーク
- **アプリケーションメトリクス**: レイテンシ、スループット、エラーレート
- **アラート履歴**: 直近のアラートと対応状況

### 経営ダッシュボード

- **KPI 概況**: ユーザー数、利用率、成長率
- **SLA 達成状況**: 可用性、応答時間の目標達成度
- **インシデント概要**: 重大インシデントとその影響
- **リソース予測**: 将来的なリソース需要予測

## 容量計画と最適化

- **使用傾向分析**: 長期的なリソース使用傾向の分析
- **自動スケーリング**: 需要に応じた自動的なリソース調整
- **コスト最適化**: 未使用リソースの特定と最適化
- **パフォーマンス分析**: ボトルネック特定と最適化提案

## 改善サイクル

1. **データ収集**: メトリクス、ログ、ユーザーフィードバック
2. **分析**: パターン認識、トレンド分析、問題点特定
3. **改善提案**: モニタリング強化、アラート最適化、パフォーマンス改善
4. **実装**: 提案の優先順位付けと実装
5. **検証**: 改善効果の測定と評価
