# コーディング規約

プロジェクトの一貫性と可読性を確保するため、以下のコーディング規約に従ってください。

詳細なフレームワーク別のガイドラインは以下のドキュメントで提供されています：

- [バックエンドコーディングルール](./standards/backend-coding-rule.md)
- [フロントエンドコーディングルール](./standards/frontend-coding-rule.md)

## 命名規則

### 共通ルール
- **定数**: `UPPER_SNAKE_CASE`
- **クラス名**: `PascalCase`
- **インターフェース名**: `PascalCase` (接頭辞 `I` は不要)
- **型名**: `PascalCase`

### フロントエンド (TypeScript/React)
- **ファイル名**: 
  - コンポーネント: `PascalCase.tsx`
  - ユーティリティ: `camelCase.ts`
  - テスト: `*.test.ts`, `*.spec.ts`
- **変数/関数**: `camelCase`
- **コンポーネント**: `PascalCase`
- **フック**: `use` + `PascalCase`
- **イベントハンドラ**: `handle` + イベント名 (例: `handleClick`)
- **プロップス型**: コンポーネント名 + `Props` (例: `ButtonProps`)

### バックエンド (Python)
- **ファイル名**: `snake_case.py`
- **変数/関数**: `snake_case`
- **クラス**: `PascalCase`
- **モジュール**: `snake_case`
- **パッケージ**: `snake_case` (ハイフンなし)
- **プライベート変数/関数**: 先頭にアンダースコア (例: `_private_method`)

## コードフォーマット

### フロントエンド
- **インデント**: 2スペース
- **最大行長**: 100文字
- **セミコロン**: 必須
- **クォート**: シングルクォート
- **末尾カンマ**: 必須（マルチライン）
- **ツール**: ESLint + Prettier

### バックエンド
- **インデント**: 4スペース
- **最大行長**: 88文字 (Black準拠)
- **ドキュメント文字列**: ダブルクォート三連符 (`"""`)
- **クォート**: ダブルクォート
- **ツール**: flake8, black, isort

## コードスタイル

### フロントエンド

#### インポート順序
```tsx
// 1. 外部ライブラリ
import React, { useState } from 'react';
import { View, Text } from 'react-native';

// 2. 内部モジュール（パスの短い順）
import { Button } from '@/shared/ui';
import { useAuth } from '@/entities/session';
import { UserProfile } from '@/features/user';

// 3. 同一ディレクトリ内のモジュール
import { styles } from './styles';
import { CONSTANTS } from './constants';
```

#### コンポーネント定義
```tsx
// 関数コンポーネントを使用
export const UserCard = ({ name, email, onPress }: UserCardProps) => {
  // 状態はフックで管理
  const [isExpanded, setIsExpanded] = useState(false);
  
  // イベントハンドラは別関数として定義
  const handleToggle = () => {
    setIsExpanded(!isExpanded);
  };
  
  return (
    <View>
      {/* JSX内の複雑なロジックは抽出 */}
      <Text>{formatUserName(name)}</Text>
      {isExpanded && <Text>{email}</Text>}
      <Button onPress={handleToggle}>
        {isExpanded ? '閉じる' : '詳細'}
      </Button>
    </View>
  );
};

// ヘルパー関数はコンポーネントの外部で定義
const formatUserName = (name: string) => {
  return name.toUpperCase();
};
```

### バックエンド

#### インポート順序
```python
# 1. 標準ライブラリ
import os
import json
from datetime import datetime
from typing import List, Optional

# 2. サードパーティライブラリ
import fastapi
from sqlmodel import select, Session
from pydantic import BaseModel

# 3. 自作モジュール
from app.core.config import settings
from app.domain.models import Patient
from app.services.auth import get_current_user
```

#### クラス/関数定義
```python
class PatientService:
    """患者関連のビジネスロジックを提供するサービス。"""
    
    def __init__(self, repository: PatientRepository):
        """
        PatientServiceを初期化します。
        
        Args:
            repository: 患者リポジトリの実装
        """
        self.repository = repository
    
    async def get_patient_by_id(self, id: str) -> Optional[Patient]:
        """
        IDに基づいて患者を取得します。
        
        Args:
            id: 患者ID
            
        Returns:
            患者オブジェクトまたはNone
        """
        return await self.repository.get_by_id(id)
```

## コメント規約

### フロントエンド
- JSDocスタイルを使用
- 関数、クラス、インターフェースには必ずコメントを追加
- 複雑なロジックには行コメントを追加

```tsx
/**
 * ユーザーの認証状態を管理するカスタムフック
 * @param initialState 初期認証状態
 * @returns 認証に関する状態と操作
 */
export const useAuth = (initialState = false) => {
  // 認証状態
  const [isAuthenticated, setIsAuthenticated] = useState(initialState);
  
  // ログイン処理
  const login = async (email: string, password: string) => {
    // ログイン処理の実装
  };
  
  return {
    isAuthenticated,
    login,
    logout: () => setIsAuthenticated(false),
  };
};
```

### バックエンド
- Googleスタイルのdocstringsを使用
- すべての公開関数、クラス、モジュールにはdocstringsを追加
- 複雑なアルゴリズムにはコメントでロジックを説明

```python
def calculate_risk_score(patient_data: dict) -> float:
    """
    患者データから健康リスクスコアを計算します。
    
    アルゴリズムは次の要素を考慮します：
    - 年齢
    - 既往歴
    - バイタルサイン
    - 生活習慣
    
    Args:
        patient_data: 患者の健康データを含む辞書
        
    Returns:
        0.0〜100.0の範囲のリスクスコア（高いほどリスクが高い）
    """
    # 基本スコアの初期化
    base_score = 0.0
    
    # 年齢に基づくスコア調整
    age = patient_data.get("age", 0)
    if age > 65:
        base_score += (age - 65) * 0.5
    
    # 他の計算ロジック...
    
    return min(100.0, base_score)
```

## コードレビュー基準

すべてのプルリクエストは、以下の基準を満たす必要があります：

1. 自動テストが全て通過していること
2. コードフォーマッターとリンターでエラーがないこと
3. 新しい機能には単体テストが書かれていること
4. セキュリティ上の問題がないこと
5. パフォーマンスに悪影響を与えないこと
6. コード規約に準拠していること
7. ドキュメントが更新されていること（必要な場合）

## FSDガイドライン（フロントエンド）

- レイヤー間の依存関係は上から下へのみ（上位レイヤーは下位レイヤーに依存可能）
- 各スライス（機能やエンティティなど）は独立して動作可能に設計
- public APIの明示的なエクスポート（index.tsを介した公開）
- レイヤー内の水平方向の依存関係は禁止
- 同じレイヤー内での共有コードは共通スライスに抽出

## DDDガイドライン（バックエンド）

- ドメインロジックはドメインレイヤーに集中させる
- エンティティと値オブジェクトを適切に分離
- ドメインサービスはドメインロジックのみを含む
- インフラストラクチャの詳細はドメインから隠蔽
- レポジトリパターンで永続化を抽象化
- 集約はトランザクション境界として使用 