# バックエンドアーキテクチャ (DDD)

バックエンドでは、Domain-Driven Design (DDD) アーキテクチャを採用し、複雑な医療ドメインを適切に表現・実装します。

## DDDの基本原則

- **ユビキタス言語**: 開発チームと医療ドメインの専門家間で共通の言語を確立し、コードとドキュメントに反映
- **境界づけられたコンテキスト**: 機能領域ごとに明確な境界を設け、各コンテキスト内でモデルの整合性を確保
- **エンティティと値オブジェクト**: データと振る舞いを持つドメインオブジェクトを明確に分離
- **集約**: 関連するエンティティと値オブジェクトをグループ化し、一貫性を保証
- **レイヤードアーキテクチャ**: ドメイン、アプリケーション、インフラストラクチャ、プレゼンテーション層の明確な分離

## ディレクトリ構造

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
```

## 実装ガイドライン

### ドメインモデルの実装

```python
# 値オブジェクト
@dataclass(frozen=True)
class PatientId:
    value: str
    
    def __post_init__(self):
        if not self.value or not isinstance(self.value, str):
            raise ValueError("患者IDは必須です")

# エンティティ
class Patient:
    def __init__(self, id: PatientId, name: str, birth_date: date):
        self.id = id
        self.name = name
        self.birth_date = birth_date
        self.medical_records: List[MedicalRecord] = []
    
    def add_medical_record(self, record: MedicalRecord) -> None:
        # ドメインロジック
        if not self.can_add_record():
            raise DomainException("患者は新しい記録を追加できません")
        self.medical_records.append(record)
        
    def can_add_record(self) -> bool:
        # ビジネスルール
        return True
```

### リポジトリインターフェース

```python
# ドメイン層のリポジトリインターフェース
class PatientRepository(Protocol):
    async def get_by_id(self, id: PatientId) -> Optional[Patient]:
        ...
    
    async def save(self, patient: Patient) -> None:
        ...
    
    async def delete(self, id: PatientId) -> None:
        ...
```

### SQLModelとの統合

```python
# インフラストラクチャ層のリポジトリ実装
class SQLModelPatientRepository:
    def __init__(self, db: Session):
        self.db = db
    
    async def get_by_id(self, id: PatientId) -> Optional[Patient]:
        patient_model = await self.db.get(PatientModel, id.value)
        if not patient_model:
            return None
        return self._map_to_domain(patient_model)
    
    async def save(self, patient: Patient) -> None:
        patient_model = PatientModel(
            id=patient.id.value,
            name=patient.name,
            birth_date=patient.birth_date
        )
        self.db.add(patient_model)
        await self.db.commit()
        
    def _map_to_domain(self, model: PatientModel) -> Patient:
        return Patient(
            id=PatientId(model.id),
            name=model.name,
            birth_date=model.birth_date
        )
```

### アプリケーションサービス

```python
# アプリケーション層のサービス
class PatientService:
    def __init__(self, repository: PatientRepository):
        self.repository = repository
    
    async def register_patient(self, name: str, birth_date: date) -> PatientId:
        # 一意のIDを生成
        patient_id = PatientId(str(uuid.uuid4()))
        
        # 患者エンティティを作成
        patient = Patient(id=patient_id, name=name, birth_date=birth_date)
        
        # リポジトリに保存
        await self.repository.save(patient)
        
        return patient_id
```

### 依存性注入

DDDアーキテクチャでは依存性注入を活用して、レイヤー間の結合度を低減します。FastAPIの依存性注入システムを使用して実装します：

```python
# 依存性プロバイダー
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_patient_repository(db: Session = Depends(get_db)):
    return SQLModelPatientRepository(db)

def get_patient_service(
    repository: PatientRepository = Depends(get_patient_repository)
):
    return PatientService(repository)
``` 