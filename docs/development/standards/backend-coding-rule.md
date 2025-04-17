# バックエンドコーディングルール

このドキュメントは、プロジェクトのバックエンド開発におけるコーディングルールの詳細を説明するものです。もともと `PROJECT_RULE.md` に含まれていた内容を、プロジェクトドキュメント構成の見直しに伴い移行・拡充したものです。

バックエンドのアーキテクチャについての詳細は [DDD アーキテクチャドキュメント](../../architecture/backend.md) を参照してください。

## 技術スタック

- **言語**: Python 3.9+
- **フレームワーク**: FastAPI
- **ORM**: SQLModel
- **マイグレーション**: Alembic
- **データベース**: PostgreSQL（JSONB 対応でリレーショナルデータと非構造化データの両方に対応）
- **認証**: Amazon Cognito（AWS SDK for Python/Boto3）
- **API 文書化**: OpenAPI/Swagger（FastAPI 組込み）
- **依存関係管理**: Poetry
- **タスクキュー**: AWS SQS + Lambda + EventBridge

## AWS 統合ガイドライン

### AWS サービス抽象化

ビジネスロジックと AWS サービスの実装を分離するために、サービス抽象化レイヤーを実装します：

```python
# app/services/queue_service.py
class QueueService:
    def __init__(self, config=None):
        self.config = config or {}
        self.is_local = self.config.get('ENV') == 'local'

        if self.is_local:
            # LocalStackまたはモックの初期化
            self.client = boto3.client(
                'sqs',
                endpoint_url='http://localhost:4566',
                region_name='us-east-1',
                aws_access_key_id='dummy',
                aws_secret_access_key='dummy'
            )
        else:
            # 本番AWSクライアント
            self.client = boto3.client('sqs')

    async def send_message(self, queue_name: str, message_data: dict) -> str:
        """メッセージをキューに送信する"""
        queue_url = self.get_queue_url(queue_name)
        response = self.client.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message_data)
        )
        return response['MessageId']

    def get_queue_url(self, queue_name: str) -> str:
        """キューのURLを取得する"""
        if self.is_local:
            return f"http://localhost:4566/000000000000/{queue_name}"
        response = self.client.get_queue_url(QueueName=queue_name)
        return response['QueueUrl']
```

### Cognito 認証の実装

```python
# app/core/auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2AuthorizationCodeBearer
import boto3
import jwt
from jwt.jwk import PyJWK
import requests

oauth2_scheme = OAuth2AuthorizationCodeBearer(
    authorizationUrl="https://your-cognito-domain/oauth2/authorize",
    tokenUrl="https://your-cognito-domain/oauth2/token",
)

class CognitoAuth:
    def __init__(self, region_name, user_pool_id, client_id):
        self.region_name = region_name
        self.user_pool_id = user_pool_id
        self.client_id = client_id
        self.jwks_url = f"https://cognito-idp.{region_name}.amazonaws.com/{user_pool_id}/.well-known/jwks.json"
        self.jwks = self._get_jwks()

    def _get_jwks(self):
        """JWKSを取得する"""
        response = requests.get(self.jwks_url)
        return response.json()

    def verify_token(self, token):
        """トークンを検証する"""
        # ヘッダーからKIDを取得
        header = jwt.get_unverified_header(token)
        kid = header['kid']

        # KIDに一致するキーを見つける
        key = None
        for jwk in self.jwks['keys']:
            if jwk['kid'] == kid:
                key = PyJWK.from_dict(jwk)
                break

        if not key:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication token"
            )

        # トークンを検証
        try:
            payload = jwt.decode(
                token,
                key.key,
                algorithms=['RS256'],
                audience=self.client_id,
                options={"verify_exp": True}
            )
            return payload
        except Exception as e:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail=f"Invalid token: {str(e)}"
            )

cognito_auth = CognitoAuth(
    region_name=settings.AWS_REGION,
    user_pool_id=settings.COGNITO_USER_POOL_ID,
    client_id=settings.COGNITO_CLIENT_ID
)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    """現在のユーザーを取得する依存性"""
    payload = cognito_auth.verify_token(token)

    # Cognitoから追加情報を取得することも可能
    # cognito_client = boto3.client('cognito-idp', region_name=settings.AWS_REGION)
    # user_info = cognito_client.get_user(AccessToken=token)

    return payload
```

### ローカル開発環境での設定

```python
# app/core/config.py
from pydantic import BaseSettings

class Settings(BaseSettings):
    # アプリケーション設定
    APP_NAME: str = "Vitascope API"
    API_V1_STR: str = "/api/v1"
    DEBUG: bool = False

    # 環境設定
    ENV: str = "dev"  # "local", "dev", "staging", "prod"

    # データベース設定
    DATABASE_URL: str

    # AWS設定
    AWS_REGION: str = "ap-northeast-1"

    # Cognito設定
    COGNITO_USER_POOL_ID: str
    COGNITO_CLIENT_ID: str

    # SQS設定
    SQS_PREFIX: str = ""  # 環境ごとのプレフィックス

    # LocalStack設定
    USE_LOCALSTACK: bool = False
    LOCALSTACK_ENDPOINT: str = "http://localhost:4566"

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

## プロジェクト構造 (DDD)

```
backend/
├── app/
│   ├── api/                     # プレゼンテーション層（FastAPIルーター）
│   │   ├── dependencies/        # 依存性注入
│   │   ├── endpoints/           # APIエンドポイント
│   │   └── routes/              # ルーティング
│   ├── application/             # アプリケーション層
│   │   ├── services/            # アプリケーションサービス
│   │   ├── dtos/                # DTOモデル
│   │   └── commands/            # コマンドハンドラ
│   ├── domain/                  # ドメイン層
│   │   ├── models/              # エンティティと値オブジェクト
│   │   ├── services/            # ドメインサービス
│   │   ├── repositories/        # リポジトリインターフェース
│   │   ├── exceptions/          # ドメイン例外
│   │   └── events/              # ドメインイベント
│   ├── infrastructure/          # インフラストラクチャ層
│   │   ├── database/            # データベース関連
│   │   ├── repositories/        # リポジトリ実装
│   │   ├── aws/                 # AWS連携
│   │   └── external/            # 外部サービス統合
│   └── core/                    # 横断的関心事
│       ├── config.py            # 環境設定
│       ├── security.py          # セキュリティ機能
│       └── events.py            # イベントハンドラ
├── alembic/                     # マイグレーション
├── tests/                       # テスト
└── pyproject.toml               # 依存関係定義
```

## データベース操作

- SQLModel のベストプラクティスに従う
- トランザクション管理の適切な実装
- N+1 問題の回避（関連データのプリフェッチ）
- インデックス最適化
- マイグレーションの適切な管理
- PostgreSQL の JSONB 機能の活用

```python
# JSONB機能の活用例
class MedicalRecord(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="user.id", index=True)
    record_type: str = Field(index=True)

    # JSONBフィールドはSQLModelでは辞書として表現
    data: Dict[str, Any] = Field(sa_column=Column(JSONB))

    # 検索用のインデックス作成
    __table_args__ = (
        Index('idx_data_gin', data, postgresql_using='gin'),
    )

# JSONBクエリの例
def search_medical_records(condition: str, db: Session) -> List[MedicalRecord]:
    # JSONB検索を使用したクエリ
    statement = select(MedicalRecord).where(
        MedicalRecord.data['condition'].astext == condition
    )
    return db.exec(statement).all()
```

## 非同期処理

- `async`/`await`の適切な使用
- I/O 集中型処理での非同期の活用
- バックグラウンドタスクの実装（AWS SQS + Lambda）
- 並列処理の最適化

```python
# AWS SQSでのバックグラウンドタスク
from app.services.queue_service import QueueService

queue_service = QueueService()

async def schedule_data_processing(user_id: int, data_type: str):
    """データ処理をキューに追加する"""
    await queue_service.send_message(
        queue_name="vitascope-data-processing",
        message_data={
            "user_id": user_id,
            "data_type": data_type,
            "timestamp": datetime.utcnow().isoformat()
        }
    )

# Lambdaハンドラー（別ファイルで定義）
def process_data_lambda_handler(event, context):
    """SQSメッセージを処理するLambda関数"""
    for record in event['Records']:
        payload = json.loads(record['body'])
        user_id = payload['user_id']
        data_type = payload['data_type']

        # データ処理ロジック
        process_user_data(user_id, data_type)

    return {'statusCode': 200}
```

## API 設計ガイドライン

バックエンド API の設計には、Google の API 設計ガイドに準拠します：

- **リソース指向設計**: API はリソースを中心に設計し、標準的な HTTP メソッドでリソースを操作
- **一貫した命名規則**: リソース名、フィールド名、メソッド名に一貫した命名規則を適用
- **標準メソッド**:
  - `GET`: リソースの取得
  - `LIST`: リソースコレクションの取得
  - `CREATE`: リソースの作成
  - `UPDATE`: リソースの更新
  - `DELETE`: リソースの削除
- **カスタムメソッド**: 標準メソッドで表現できない操作には`:カスタムアクション`形式を使用
- **エラー処理**: 標準的なエラーコードとメッセージ形式を使用
- **バージョニング**: API バージョンは URI パスに含める（例: `/v1/resources`）

詳細な API 設計ガイドラインについては [API 設計ガイド](../api-guidelines.md) を参照してください。

## セキュリティとコンプライアンス戦略

### セキュリティ対策

- **データ保護**:
  - 保存データの暗号化(AES-256)
  - 転送中データの TLS 1.3
  - PHI（個人特定可能な健康情報）の匿名化

```python
# 推奨: データ暗号化ユーティリティ
from cryptography.fernet import Fernet
from app.core.config import settings

class Encryptor:
    def __init__(self):
        self.key = settings.ENCRYPTION_KEY.encode()
        self.cipher = Fernet(self.key)

    def encrypt(self, data: str) -> str:
        """データを暗号化する"""
        return self.cipher.encrypt(data.encode()).decode()

    def decrypt(self, encrypted_data: str) -> str:
        """暗号化されたデータを復号する"""
        return self.cipher.decrypt(encrypted_data.encode()).decode()

# 推奨: 個人健康情報（PHI）フィールドの暗号化
class PatientRecord(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="user.id")

    # 暗号化フィールド
    _encrypted_medical_history: str = Field(sa_column=Column(String(1024)))

    # プロパティを使用した自動暗号化/復号
    @property
    def medical_history(self) -> str:
        encryptor = Encryptor()
        return encryptor.decrypt(self._encrypted_medical_history)

    @medical_history.setter
    def medical_history(self, value: str):
        encryptor = Encryptor()
        self._encrypted_medical_history = encryptor.encrypt(value)
```

プロジェクト全体のセキュリティポリシーに準拠し、実装する必要があります。詳細は [セキュリティポリシー](../../operations/security.md) を参照してください。

### 監視・ログ戦略

```python
# app/core/logging.py
import logging
import json
from pythonjsonlogger import jsonlogger
from app.core.config import settings

# 構造化ログフォーマッター
class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super(CustomJsonFormatter, self).add_fields(log_record, record, message_dict)
        log_record['app'] = settings.APP_NAME
        log_record['environment'] = settings.ENV
        log_record['level'] = record.levelname

        # リクエストIDがある場合は追加
        if hasattr(record, 'request_id'):
            log_record['request_id'] = record.request_id

# ロガー設定
def setup_logging():
    logger = logging.getLogger("app")

    # ログレベル設定
    log_level = logging.DEBUG if settings.DEBUG else logging.INFO
    logger.setLevel(log_level)

    # ハンドラー設定
    handler = logging.StreamHandler()
    formatter = CustomJsonFormatter('%(timestamp)s %(level)s %(name)s %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger

# センシティブデータマスク関数
def mask_sensitive_data(data: dict) -> dict:
    """センシティブデータをマスクする"""
    if not data:
        return data

    masked = data.copy()
    sensitive_fields = ['password', 'ssn', 'credit_card', 'medical_id']

    for field in sensitive_fields:
        if field in masked:
            masked[field] = '******'

    # メールアドレスの部分マスク
    if 'email' in masked and isinstance(masked['email'], str):
        parts = masked['email'].split('@')
        if len(parts) == 2:
            username = parts[0]
            domain = parts[1]
            if len(username) > 2:
                masked_username = username[0:2] + '*' * (len(username) - 2)
                masked['email'] = f"{masked_username}@{domain}"

    return masked
```

詳細な監視戦略とアラート設定については [監視戦略](../../operations/monitoring.md) を参照してください。

## テスト戦略

品質を確保するための包括的なテスト戦略を実装します。

### テスト種類と対象

- **単体テスト** (カバレッジ目標: 80%以上)

  - ビジネスロジック
  - ユーティリティ関数
  - モデルとデータ変換

- **統合テスト** (カバレッジ目標: 60%以上)

  - API エンドポイント
  - データベースとの連携
  - 外部サービス連携

- **E2E テスト**
  - 主要ユーザーフローの自動テスト
  - クリティカルパスの定期的テスト
  - パフォーマンステスト

### AWS 統合テスト

```python
# test_aws_services.py
import pytest
from unittest.mock import patch, MagicMock
from app.services.queue_service import QueueService

# AWS SDKのモック化
@pytest.fixture
def mock_sqs_client():
    with patch("boto3.client") as mock_client:
        mock_sqs = MagicMock()
        mock_sqs.send_message.return_value = {"MessageId": "test-message-id"}
        mock_sqs.get_queue_url.return_value = {"QueueUrl": "http://test-queue-url"}
        mock_client.return_value = mock_sqs
        yield mock_sqs

# QueueServiceのテスト
async def test_send_message(mock_sqs_client):
    # テスト用インスタンス
    service = QueueService(config={"ENV": "test"})

    # メッセージ送信
    message_id = await service.send_message(
        queue_name="test-queue",
        message_data={"test": "data"}
    )

    # アサーション
    assert message_id == "test-message-id"
    mock_sqs_client.send_message.assert_called_once_with(
        QueueUrl="http://test-queue-url",
        MessageBody='{"test": "data"}'
    )
```

## レビューチェックリスト

- [ ] Python コーディングスタイル（PEP 8）に従っているか
- [ ] 型ヒントは適切に使用されているか
- [ ] エラーハンドリングは適切か
- [ ] セキュリティ対策は考慮されているか
- [ ] AWS 抽象化レイヤーを適切に使用しているか
- [ ] ローカル開発環境での動作確認は行われているか
- [ ] パフォーマンス最適化は考慮されているか
- [ ] テストは十分に書かれているか（カバレッジ目標達成）
- [ ] ドキュメントは更新されているか
- [ ] 医療データ規格に準拠しているか
- [ ] 非同期処理は適切に使用されているか
- [ ] 依存性注入の原則は守られているか
- [ ] コンプライアンス要件を満たしているか

## 基本設計原則 (DDD)

バックエンドは Domain-Driven Design (DDD) の原則に従って設計・実装します。

### レイヤードアーキテクチャ

```
presentation(API) → application → domain ← infrastructure
```

- **プレゼンテーション層**: ユーザーとシステムの相互作用（API エンドポイント、レスポンス形式など）
- **アプリケーション層**: ユースケースを表現し、ドメイン層を調整
- **ドメイン層**: ビジネスルール、エンティティ、値オブジェクトなどのドメインモデル
- **インフラストラクチャ層**: データベース、外部サービス連携、リポジトリの実装など

### 典型的な DDD の実装例

```python
# ドメイン層 - 値オブジェクト
@dataclass(frozen=True)
class PatientId:
    """患者を一意に識別するID"""
    value: str

    def __post_init__(self):
        """IDの値が有効かを検証"""
        if not self.value or not isinstance(self.value, str):
            raise ValueError("患者IDは空にできません")
        if not re.match(r'^[A-Z]{2}\d{6}$', self.value):
            raise ValueError("患者IDの形式が無効です")

# ドメイン層 - エンティティ
class Patient:
    """患者を表すエンティティ"""
    def __init__(self, id: PatientId, name: str, birth_date: date):
        self.id = id
        self.name = name
        self.birth_date = birth_date
        self._medical_records: List[MedicalRecord] = []

    @property
    def medical_records(self) -> List[MedicalRecord]:
        """医療記録のコピーを返し、直接変更を防止"""
        return self._medical_records.copy()

    def add_medical_record(self, record: MedicalRecord) -> None:
        """
        医療記録を追加する。患者の状態に基づいて追加可能かを検証。

        Args:
            record: 追加する医療記録

        Raises:
            DomainException: 記録を追加できない場合
        """
        if not self.can_add_record(record):
            raise DomainException("患者は現在の状態では新しい記録を追加できません")
        self._medical_records.append(record)

    def can_add_record(self, record: MedicalRecord) -> bool:
        """
        患者が医療記録を追加可能かを検証するビジネスルール

        Args:
            record: 追加予定の医療記録

        Returns:
            追加可能かどうかのブール値
        """
        # ビジネスルールのチェック例
        return True

# ドメイン層 - リポジトリインターフェース
class PatientRepository(Protocol):
    """患者データの永続化を担当するリポジトリのインターフェース"""

    async def get_by_id(self, id: PatientId) -> Optional[Patient]:
        """IDから患者を取得する"""
        ...

    async def save(self, patient: Patient) -> None:
        """患者を保存する"""
        ...

    async def find_by_criteria(self, criteria: dict) -> List[Patient]:
        """条件に合致する患者を検索する"""
        ...

# インフラストラクチャ層 - リポジトリ実装
class SQLAlchemyPatientRepository:
    """SQLAlchemyを使用した患者リポジトリの実装"""

    def __init__(self, db: Session):
        self.db = db

    async def get_by_id(self, id: PatientId) -> Optional[Patient]:
        """
        指定されたIDの患者をデータベースから取得

        Args:
            id: 検索する患者ID

        Returns:
            患者エンティティ、または該当なしの場合None
        """
        patient_dto = await self.db.query(PatientModel).filter(
            PatientModel.id == id.value
        ).first()
        if not patient_dto:
            return None
        return self._map_to_entity(patient_dto)

    async def save(self, patient: Patient) -> None:
        """
        患者エンティティをデータベースに保存

        Args:
            patient: 保存する患者エンティティ
        """
        patient_dto = await self.db.query(PatientModel).filter(
            PatientModel.id == patient.id.value
        ).first()

        if patient_dto:
            # 既存レコードの更新
            patient_dto.name = patient.name
            patient_dto.birth_date = patient.birth_date
        else:
            # 新規レコードの作成
            patient_dto = PatientModel(
                id=patient.id.value,
                name=patient.name,
                birth_date=patient.birth_date
            )
            self.db.add(patient_dto)

        await self.db.commit()

    def _map_to_entity(self, dto: PatientModel) -> Patient:
        """DTOからドメインエンティティへのマッピング"""
        return Patient(
            id=PatientId(dto.id),
            name=dto.name,
            birth_date=dto.birth_date
        )

# アプリケーション層 - サービス
class PatientService:
    """患者に関するユースケースを実装するアプリケーションサービス"""

    def __init__(self, repository: PatientRepository):
        self.repository = repository

    async def register_patient(self, name: str, birth_date: date) -> PatientId:
        """
        新しい患者を登録するユースケース

        Args:
            name: 患者名
            birth_date: 生年月日

        Returns:
            登録された患者のID
        """
        # IDの生成
        patient_id = PatientId(f"PT{random.randint(100000, 999999)}")

        # 患者エンティティの作成
        patient = Patient(id=patient_id, name=name, birth_date=birth_date)

        # リポジトリに保存
        await self.repository.save(patient)

        return patient_id

    async def get_patient_details(self, patient_id: PatientId) -> Optional[PatientDetailsDTO]:
        """
        患者の詳細情報を取得するユースケース

        Args:
            patient_id: 患者ID

        Returns:
            患者詳細DTO、または該当なしの場合None
        """
        patient = await self.repository.get_by_id(patient_id)
        if not patient:
            return None

        return PatientDetailsDTO(
            id=patient.id.value,
            name=patient.name,
            birth_date=patient.birth_date,
            medical_record_count=len(patient.medical_records)
        )

# プレゼンテーション層 - APIエンドポイント
@router.post("/patients", response_model=PatientResponse)
async def create_patient(
    request: PatientCreateRequest,
    service: PatientService = Depends(get_patient_service)
):
    """
    新規患者を登録するエンドポイント

    Args:
        request: 患者登録リクエスト
        service: 依存性注入された患者サービス

    Returns:
        登録された患者情報
    """
    try:
        # ユースケースの実行
        patient_id = await service.register_patient(
            name=request.name,
            birth_date=request.birth_date
        )

        # レスポンスの構築
        return PatientResponse(id=patient_id.value, status="success")
    except ValueError as e:
        # バリデーションエラー
        raise HTTPException(status_code=400, detail=str(e))
    except DomainException as e:
        # ドメインロジックによるエラー
        raise HTTPException(status_code=422, detail=str(e))
    except Exception as e:
        # その他の予期せぬエラー
        logger.error(f"患者登録中にエラーが発生しました: {e}")
        raise HTTPException(status_code=500, detail="Internal Server Error")
```

## Python コーディング規約

### 命名規則

- **パッケージ名**: `snake_case`
- **モジュール名**: `snake_case`
- **クラス名**: `PascalCase`
- **例外名**: `PascalCase` (通常 `Error` または `Exception` で終わる)
- **関数名**: `snake_case`
- **変数名**: `snake_case`
- **定数名**: `UPPER_SNAKE_CASE`
- **プライベート関数/変数**: `_leading_underscore`
- **"マジック"メソッド**: `__double_leading_and_trailing_underscores__`

### インデントとフォーマット

- **インデント**: 4 スペース
- **行の長さ**: 最大 88 文字 (PEP 8 + Black に準拠)
- **import 文**:
  - 標準ライブラリ
  - サードパーティライブラリ
  - アプリケーション固有のインポート
  - それぞれのグループ間には空行を入れる
- **クォート**: ダブルクォート `"` を優先

### コメントとドキュメンテーション

fastAPI API エンドポイントでは、すべての関数に適切な Docstring を提供します。これは OpenAPI ドキュメントの生成に使用されます。以下の Google スタイルを採用します。

```python
def function_with_types_in_docstring(param1: str, param2: int) -> bool:
    """リクエストを検証して結果を返す関数です。

    この関数は、提供されたパラメータが有効かどうかを確認し、
    結果に基づいてブール値を返します。

    Args:
        param1: 最初のパラメータの説明
        param2: 2番目のパラメータの説明。
               複数行の説明の場合はインデントを合わせます。

    Returns:
        検証結果。True は成功、False は失敗を意味します。

    Raises:
        ValueError: 無効なパラメータが提供された場合
        TypeError: パラメータの型が誤っている場合

    Examples:
        >>> function_with_types_in_docstring("test", 1)
        True
    """
    if param1 == "invalid":
        raise ValueError("param1は'invalid'にできません")
    return True
```

### 例外処理

明示的な型付き例外を使用し、適切なエラーメッセージを提供します：

```python
try:
    result = potentially_failing_function()
except ValueError as e:
    # 特定の例外を処理
    logger.warning(f"無効な値が検出されました: {e}")
    raise HTTPException(status_code=400, detail=str(e))
except Exception as e:
    # 予期せぬ例外をログに記録し、ユーザーに一般的なエラーを表示
    logger.error(f"予期せぬエラーが発生しました: {e}", exc_info=True)
    raise HTTPException(status_code=500, detail="内部サーバーエラー")
```

裸の `except:` や `except Exception:` は避け、特定の例外を捕捉するようにします。

### FastAPI に関する規約

#### 依存性注入

依存関係には、Depends を使用します：

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_patient_repository(db = Depends(get_db)):
    return SQLAlchemyPatientRepository(db)

def get_patient_service(repo = Depends(get_patient_repository)):
    return PatientService(repo)

@app.get("/patients/{patient_id}")
async def get_patient(
    patient_id: str,
    service: PatientService = Depends(get_patient_service)
):
    return await service.get_patient_details(PatientId(patient_id))
```

#### Pydantic モデル

リクエスト/レスポンスのバリデーションとパースには Pydantic モデルを使用します：

```python
from pydantic import BaseModel, Field, validator
from datetime import date
from typing import Optional, List

class PatientCreateRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100, example="山田太郎")
    birth_date: date = Field(..., example="1980-01-01")
    gender: str = Field(..., example="male")

    @validator('gender')
    def validate_gender(cls, v):
        allowed = ['male', 'female', 'other']
        if v not in allowed:
            raise ValueError(f"性別は {', '.join(allowed)} のいずれかである必要があります")
        return v

class PatientResponse(BaseModel):
    id: str
    status: str

    class Config:
        schema_extra = {
            "example": {
                "id": "PT123456",
                "status": "success"
            }
        }
```

## テスト規約

### 単体テスト

PyTest を使用してテストを書き、適切な命名規則とディレクトリ構造に従います：

```python
# tests/unit/domain/test_patient.py
import pytest
from datetime import date
from app.domain.models import Patient, PatientId, DomainException

def test_create_patient():
    # Arrange
    patient_id = PatientId("PT123456")
    name = "山田太郎"
    birth_date = date(1980, 1, 1)

    # Act
    patient = Patient(id=patient_id, name=name, birth_date=birth_date)

    # Assert
    assert patient.id == patient_id
    assert patient.name == name
    assert patient.birth_date == birth_date
    assert len(patient.medical_records) == 0

def test_add_medical_record():
    # Arrange
    patient = Patient(
        id=PatientId("PT123456"),
        name="山田太郎",
        birth_date=date(1980, 1, 1)
    )
    record = MedicalRecord("発熱", date.today())

    # Act
    patient.add_medical_record(record)

    # Assert
    assert len(patient.medical_records) == 1
    assert patient.medical_records[0] == record

def test_invalid_patient_id():
    # Arrange & Act & Assert
    with pytest.raises(ValueError, match="患者IDの形式が無効です"):
        PatientId("invalid")
```

### 統合テスト

FastAPI の testclient を使用して API エンドポイントをテストします：

```python
# tests/integration/api/test_patient_api.py
from fastapi.testclient import TestClient
from app.main import app
import pytest

client = TestClient(app)

def test_create_patient_success():
    # Arrange
    request_data = {
        "name": "山田太郎",
        "birth_date": "1980-01-01",
        "gender": "male"
    }

    # Act
    response = client.post("/api/v1/patients", json=request_data)

    # Assert
    assert response.status_code == 200
    data = response.json()
    assert "id" in data
    assert data["status"] == "success"

def test_create_patient_invalid_data():
    # Arrange
    request_data = {
        "name": "",  # 空の名前は無効
        "birth_date": "1980-01-01",
        "gender": "male"
    }

    # Act
    response = client.post("/api/v1/patients", json=request_data)

    # Assert
    assert response.status_code == 422  # バリデーションエラー
```

### モック

依存関係をモックしてユニットテストを分離します：

```python
# tests/unit/application/test_patient_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from datetime import date
from app.application.services import PatientService
from app.domain.models import Patient, PatientId

@pytest.fixture
def mock_repository():
    repo = AsyncMock()
    # 必要なメソッドのモック動作を設定
    return repo

def test_register_patient(mock_repository):
    # Arrange
    service = PatientService(repository=mock_repository)
    name = "山田太郎"
    birth_date = date(1980, 1, 1)

    # Expectation
    mock_repository.save.return_value = None

    # Act
    patient_id = await service.register_patient(name, birth_date)

    # Assert
    assert isinstance(patient_id, PatientId)
    mock_repository.save.assert_called_once()
    # 保存された患者オブジェクトの検証
    saved_patient = mock_repository.save.call_args[0][0]
    assert saved_patient.name == name
    assert saved_patient.birth_date == birth_date
```

## ログ記録規約

Python 標準の`logging`モジュールを使用し、構造化ログを提供します：

```python
import logging
import json
from datetime import datetime

logger = logging.getLogger(__name__)

def log_request(request_id, user_id, action, data=None):
    """リクエストを構造化形式でログに記録"""
    log_data = {
        "timestamp": datetime.utcnow().isoformat(),
        "request_id": request_id,
        "user_id": user_id,
        "action": action,
        "data": data
    }
    logger.info(json.dumps(log_data))

# 使用例
@app.post("/api/v1/patients")
async def create_patient(
    request: PatientCreateRequest,
    current_user = Depends(get_current_user)
):
    request_id = str(uuid.uuid4())
    log_request(request_id, current_user.id, "create_patient", request.dict())
    # 処理続行...
```

## パフォーマンスのベストプラクティス

- データベースクエリの最適化（N+1 問題の回避）
- 大量データ処理時のページネーション実装
- 適切なキャッシング戦略
- 非同期処理の活用（FastAPI の async/await）

```python
# 効率的なデータベースクエリの例
async def get_patients_with_records(limit: int = 100, offset: int = 0):
    # N+1問題を避けるためにJOINを使用
    query = (
        select(PatientModel, MedicalRecordModel)
        .join(MedicalRecordModel, PatientModel.id == MedicalRecordModel.patient_id)
        .limit(limit)
        .offset(offset)
    )

    result = await database.fetch_all(query)
    # 結果の処理...
```

## セキュリティのベストプラクティス

- 入力データの厳密な検証
- SQL インジェクション対策（ORM、パラメータクエリの使用）
- XSS 対策（出力エンコーディング）
- 適切な認証・認可メカニズム（JWT、OAuth2）
- 機密データの暗号化

```python
# 機密データの暗号化例
from cryptography.fernet import Fernet

def encrypt_sensitive_data(data: str) -> str:
    key = settings.ENCRYPTION_KEY
    f = Fernet(key)
    return f.encrypt(data.encode()).decode()

def decrypt_sensitive_data(encrypted_data: str) -> str:
    key = settings.ENCRYPTION_KEY
    f = Fernet(key)
    return f.decrypt(encrypted_data.encode()).decode()
```
