# API 設計ガイドライン

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

## 認証と認可

API 認証には Amazon Cognito を使用し、JWT トークンベースの認証を実装します。

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="認証情報が無効です",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        # トークン検証ロジック
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    # ユーザー情報の取得
    user = await get_user(user_id)
    if user is None:
        raise credentials_exception
    return user

# APIエンドポイントでの使用例
@router.get("/users/me", response_model=UserResponse)
async def get_current_user_info(current_user: User = Depends(get_current_user)):
    return current_user
```

### 認可レベル

- **PUBLIC**: 認証不要のエンドポイント（ヘルスチェック、公開情報など）
- **USER**: 一般ユーザー向けエンドポイント（個人データアクセスなど）
- **PROVIDER**: 医療提供者向けエンドポイント（患者データアクセスなど）
- **ADMIN**: 管理者向けエンドポイント（システム管理など）

## FastAPI による実装

```python
# GoogleのAPI設計ガイドに準拠したFastAPI実装例
from fastapi import APIRouter, Path, Query, HTTPException
from pydantic import BaseModel
from typing import List, Optional

router = APIRouter(prefix="/v1")

class UserResponse(BaseModel):
    id: str
    name: str
    email: str

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: str = Path(...)):
    # リソース取得ロジック
    return {"id": user_id, "name": "山田太郎", "email": "yamada@example.com"}

@router.get("/users", response_model=List[UserResponse])
async def list_users(
    page_size: int = Query(50, gt=0),
    page_token: Optional[str] = None
):
    # コレクション取得ロジック
    return [{"id": "1", "name": "山田太郎", "email": "yamada@example.com"}]

@router.post("/users/{user_id}:sendVerification")
async def send_verification(user_id: str):
    # カスタムアクション実行
    return {"result": "送信完了"}
```

## エラーハンドリング

標準的な HTTP ステータスコードとレスポンス形式を使用します：

```python
# エラーレスポンスモデル
class ErrorResponse(BaseModel):
    code: int
    message: str
    details: Optional[List[dict]] = None

# エラーハンドリングの例
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "code": exc.status_code,
            "message": exc.detail,
            "details": exc.headers.get("details")
        },
    )
```

## API ドキュメント自動生成

FastAPI の組み込み機能を活用して、OpenAPI ドキュメントを自動生成します：

- Swagger UI: `/docs` エンドポイントで利用可能
- ReDoc: `/redoc` エンドポイントで利用可能
- OpenAPI JSON: `/openapi.json` エンドポイントで利用可能

全ての API エンドポイントには適切なドキュメンテーションコメントを付与し、モデルには明確な説明を含めます。
