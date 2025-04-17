# フロントエンドコーディングルール

## 技術スタック

- **フレームワーク**: React Native
- **開発環境**: Expo
- **状態管理**: Redux Toolkit
- **スタイリング**: NativeWind（TailwindベースのRN向けスタイリング）
- **アーキテクチャ**: Feature-Sliced Design (FSD)
- **認証**: Amazon Cognito SDK

## AWS サービス連携

### Cognito認証の実装

```tsx
// shared/lib/auth/cognito.ts
import {
  CognitoUserPool,
  CognitoUser,
  AuthenticationDetails,
  CognitoUserSession
} from 'amazon-cognito-identity-js';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { CognitoConfig } from '@/shared/config';

// Cognito設定
const userPool = new CognitoUserPool({
  UserPoolId: CognitoConfig.USER_POOL_ID,
  ClientId: CognitoConfig.CLIENT_ID,
  Storage: AsyncStorage
});

export class CognitoService {
  // サインイン
  static signIn(email: string, password: string): Promise<CognitoUserSession> {
    return new Promise((resolve, reject) => {
      const authDetails = new AuthenticationDetails({
        Username: email,
        Password: password
      });
      
      const cognitoUser = new CognitoUser({
        Username: email,
        Pool: userPool,
        Storage: AsyncStorage
      });
      
      cognitoUser.authenticateUser(authDetails, {
        onSuccess: (session) => {
          resolve(session);
        },
        onFailure: (err) => {
          reject(err);
        },
        newPasswordRequired: (userAttributes) => {
          // 初回ログイン時のパスワード変更が必要な場合
          reject(new Error('NEW_PASSWORD_REQUIRED'));
        }
      });
    });
  }
  
  // サインアップ
  static signUp(email: string, password: string, attributes: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const attributeList = Object.keys(attributes).map(key => ({
        Name: key,
        Value: attributes[key]
      }));
      
      userPool.signUp(email, password, attributeList, null, (err, result) => {
        if (err) {
          reject(err);
          return;
        }
        resolve(result);
      });
    });
  }
  
  // トークン取得
  static async getTokens(): Promise<{
    idToken: string;
    accessToken: string;
    refreshToken: string;
  } | null> {
    return new Promise((resolve, reject) => {
      const cognitoUser = userPool.getCurrentUser();
      
      if (!cognitoUser) {
        resolve(null);
        return;
      }
      
      cognitoUser.getSession((err: Error | null, session: CognitoUserSession | null) => {
        if (err) {
          reject(err);
          return;
        }
        
        if (!session) {
          resolve(null);
          return;
        }
        
        resolve({
          idToken: session.getIdToken().getJwtToken(),
          accessToken: session.getAccessToken().getJwtToken(),
          refreshToken: session.getRefreshToken().getToken()
        });
      });
    });
  }
  
  // ログアウト
  static signOut(): void {
    const cognitoUser = userPool.getCurrentUser();
    if (cognitoUser) {
      cognitoUser.signOut();
    }
  }
}
```

### AWS サービス抽象化レイヤー

```tsx
// shared/lib/api/api.service.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { CognitoService } from '@/shared/lib/auth/cognito';
import { API_CONFIG } from '@/shared/config';

export class ApiService {
  private static _instance: AxiosInstance;
  
  static get instance(): AxiosInstance {
    if (!this._instance) {
      this._instance = axios.create({
        baseURL: API_CONFIG.BASE_URL,
        timeout: 10000,
        headers: {
          'Content-Type': 'application/json'
        }
      });
      
      // リクエストインターセプター（トークン付与）
      this._instance.interceptors.request.use(
        async (config) => {
          const tokens = await CognitoService.getTokens();
          if (tokens) {
            config.headers['Authorization'] = `Bearer ${tokens.idToken}`;
          }
          return config;
        },
        (error) => Promise.reject(error)
      );
      
      // レスポンスインターセプター（エラーハンドリング）
      this._instance.interceptors.response.use(
        (response) => response,
        async (error) => {
          // 401エラー時に再認証を試みる
          if (error.response && error.response.status === 401) {
            try {
              // TODO: トークンリフレッシュロジック
              // const newTokens = await refreshTokens();
              // リクエストを再試行
              // return this._instance(error.config);
            } catch (refreshError) {
              // リフレッシュ失敗時はログアウト
              CognitoService.signOut();
            }
          }
          return Promise.reject(error);
        }
      );
    }
    
    return this._instance;
  }
}
```

## LocalStack連携開発

ローカル開発環境でAWSサービスをエミュレートするためのガイドライン：

```tsx
// shared/config/aws.config.ts
import { Platform } from 'react-native';
import { Constants } from 'expo-constants';

const ENV = process.env.NODE_ENV || 'development';

// LocalStackのエンドポイント（開発環境のみ）
const LOCALSTACK_ENDPOINT = Platform.select({
  // Android Emulatorの場合
  android: 'http://10.0.2.2:4566',
  // iOS Simulatorの場合
  ios: 'http://localhost:4566',
  // その他
  default: 'http://localhost:4566'
});

export const AWS_CONFIG = {
  // 開発環境ではLocalStackを使用
  useLocalStack: ENV === 'development',
  // LocalStackのエンドポイント
  localStackEndpoint: LOCALSTACK_ENDPOINT,
  // リージョン
  region: 'ap-northeast-1',
  // Cognito設定
  cognito: {
    userPoolId: ENV === 'development' ? 'local_user_pool' : process.env.COGNITO_USER_POOL_ID,
    clientId: ENV === 'development' ? 'local_client_id' : process.env.COGNITO_CLIENT_ID,
  }
};
```

## プロジェクト構造 (FSD)

FSDアーキテクチャに基づく階層構造を採用しています：

```
src/
├── app/        # アプリケーション初期化、プロバイダー、ルーティング
├── processes/  # クロスフィーチャーのワークフロー、グローバルプロセス
├── screens/    # 画面コンポーネント（Webアプリにおけるpagesに相当）
├── widgets/    # 複合UIブロック、画面の独立した部分
├── features/   # ユーザー操作に関連する機能単位
├── entities/   # ビジネスエンティティ（患者、医療記録など）
└── shared/     # 共通ユーティリティ、UIキット、API、型定義など
    ├── api/        # API連携
    ├── config/     # 環境設定
    ├── lib/        # ユーティリティ
    │   ├── auth/   # 認証関連
    │   ├── aws/    # AWS連携
    │   └── hooks/  # カスタムフック
    └── ui/         # 共通UIコンポーネント
```

## コード規約

### 命名規則

- **ファイル名**:
  - コンポーネント: `PascalCase.tsx`
  - ユーティリティ: `camelCase.ts`
  - テスト: `ComponentName.test.tsx`
  - スタイル: `ComponentName.styles.ts`

- **変数/関数**: camelCase
- **コンポーネント**: PascalCase
- **インターフェース/型**: PascalCase, 接頭辞なし (例: `User` not `IUser`)
- **定数**: UPPER_SNAKE_CASE

### NativeWindスタイリングガイドライン

- インラインスタイルよりもクラス名を優先
- 共通スタイルは`shared/ui`に定義
- 複雑なスタイルはカスタムクラスとして分離
- スタイルの衝突を避けるためにスコープを考慮

```tsx
// 推奨
function Button({ children }) {
  return (
    <TouchableOpacity className="bg-blue-500 px-4 py-2 rounded-md">
      <Text className="text-white font-medium">{children}</Text>
    </TouchableOpacity>
  );
}

// 非推奨
function Button({ children }) {
  return (
    <TouchableOpacity style={{ backgroundColor: 'blue', padding: 10 }}>
      <Text style={{ color: 'white' }}>{children}</Text>
    </TouchableOpacity>
  );
}
```

### コンポーネント設計

- 単一責任の原則に従う
- Presentational/Container パターンの活用
- 再利用可能なコンポーネントは`shared/ui`に配置
- スクリーン固有のコンポーネントは対応するスクリーンディレクトリに配置
- 適切なmemoization（React.memo, useMemo, useCallback）の使用

```tsx
// 推奨: 関心の分離
function UserProfileScreen() {
  const { userData, isLoading } = useUserData();
  
  if (isLoading) return <LoadingIndicator />;
  
  return <UserProfileCard user={userData} />;
}

// 非推奨: 責任が混在
function UserProfileScreen() {
  const [userData, setUserData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    fetchUserData()
      .then(data => {
        setUserData(data);
        setIsLoading(false);
      });
  }, []);
  
  if (isLoading) return <ActivityIndicator />;
  
  return (
    <View>
      <Text>{userData.name}</Text>
      <Text>{userData.email}</Text>
      {/* 他の表示ロジック */}
    </View>
  );
}
```

### FSD実装ガイドライン

- レイヤー間の依存関係は上から下へのみ（上位レイヤーは下位レイヤーに依存可能）
- 各スライス（機能やエンティティなど）は独立して動作するよう設計
- public APIの明示的なエクスポート：各スライスのルートに`index.ts`を配置

```tsx
// features/auth/index.ts
// Public API
export { LoginForm } from './ui/LoginForm';
export { useAuth } from './model/useAuth';
export type { AuthState } from './model/types';

// features/auth/ui/LoginForm.tsx内で直接他の機能をインポートしない
// 非推奨: import { ProfileCard } from '../../profile/ui/ProfileCard';
```

### 状態管理

- Redux Toolkitを使用したReduxストアの実装
- スライスは機能ごとに分割
- 非同期ロジックにはRedux Thunkを使用
- セレクターの積極的な活用

```tsx
// 推奨: 機能ごとに分割されたスライス
// features/auth/model/slice.ts
import { createSlice } from '@reduxjs/toolkit';

export const authSlice = createSlice({
  name: 'auth',
  initialState: { /* ... */ },
  reducers: { /* ... */ }
});

// features/healthMetrics/model/slice.ts
import { createSlice } from '@reduxjs/toolkit';

export const healthMetricsSlice = createSlice({
  name: 'healthMetrics',
  initialState: { /* ... */ },
  reducers: { /* ... */ }
});
```

### TypeScript活用

- `any`型の使用を避ける
- 適切なインターフェース定義
- ユーティリティ型の活用
- 厳格なnullチェック
- Props型の明示的定義

```tsx
// 推奨
interface UserProfileProps {
  userId: string;
  showDetails?: boolean;
}

function UserProfile({ userId, showDetails = false }: UserProfileProps) {
  // ...
}

// 非推奨
function UserProfile(props) {
  const { userId, showDetails = false } = props;
  // ...
}
```

### パフォーマンス最適化

- 不要な再レンダリングの回避
- 大きなリストには`FlatList`の使用
- 画像の適切な最適化
- メモ化の適切な活用
- コンポーネントの遅延ロード

### セキュリティ考慮事項

- ユーザー入力の検証
- 機密データの安全な保存（Expo SecureStore）
- APIキーや認証情報のソースコードへの埋め込み禁止
- 入力インジェクション対策

## レビューチェックリスト

- [ ] FSDアーキテクチャの原則に従っているか
- [ ] TypeScriptの型定義は適切か
- [ ] 命名規則に従っているか
- [ ] AWS認証が適切に実装されているか
- [ ] オフラインサポートは考慮されているか
- [ ] エラー監視とレポートが実装されているか
- [ ] テストは十分に書かれているか（カバレッジ目標達成）
- [ ] パフォーマンスを考慮しているか
- [ ] UIコンポーネントはアクセシビリティに配慮しているか
- [ ] セキュリティ対策は考慮されているか
- [ ] 医療データ規制に準拠しているか
- [ ] コード分割とバンドルサイズの最適化は行われているか

## デバッグとテスト

- Jest + React Native Testing Libraryを使用したユニットテスト
- Detoxを使用したE2Eテスト
- Expo開発者ツールの活用
- React DevToolsの活用
- Reduxデバッグツールの活用

## 医療データ処理のガイドライン

- 個人特定可能な医療情報（PHI）は暗号化して保存
- APIレスポンスに含まれる医療データは適切に型付け
- 医療データ表示時の単位や参照範囲を明示
- オフライン時の医療データ同期戦略を考慮
- 医療用語や診断コードは標準規格（FHIR, HL7, SNOMED CT等）を使用

## 監視・アラート実装

### エラー監視

アプリケーションのエラーを効率的に収集・分析するための実装：

```tsx
// shared/lib/monitoring/error-tracking.ts
import * as Sentry from 'sentry-expo';
import { APP_CONFIG } from '@/shared/config';

export class ErrorTracker {
  static init() {
    if (APP_CONFIG.ENV !== 'development') {
      Sentry.init({
        dsn: APP_CONFIG.SENTRY_DSN,
        enableInExpoDevelopment: false,
        debug: APP_CONFIG.ENV === 'staging',
        environment: APP_CONFIG.ENV
      });
    }
  }
  
  static captureException(error: Error, context?: Record<string, any>) {
    console.error(error);
    
    if (APP_CONFIG.ENV !== 'development') {
      Sentry.Native.captureException(error, {
        extra: context
      });
    }
  }
  
  static setUser(userId: string | null, additionalData?: Record<string, any>) {
    if (APP_CONFIG.ENV !== 'development') {
      if (userId) {
        Sentry.Native.setUser({
          id: userId,
          ...additionalData
        });
      } else {
        Sentry.Native.setUser(null);
      }
    }
  }
}
```

### パフォーマンス監視

アプリケーションのパフォーマンスを測定するための実装：

```tsx
// shared/lib/monitoring/performance.ts
import { useState, useEffect } from 'react';
import { InteractionManager } from 'react-native';
import { ErrorTracker } from './error-tracking';

// 画面レンダリング時間の計測
export function useScreenLoadTime(screenName: string) {
  useEffect(() => {
    const startTime = Date.now();
    
    // インタラクションが完了したタイミングで計測
    InteractionManager.runAfterInteractions(() => {
      const loadTime = Date.now() - startTime;
      console.log(`Screen load time (${screenName}): ${loadTime}ms`);
      
      // 閾値を超えた場合は警告
      if (loadTime > 1000) {
        ErrorTracker.captureException(
          new Error(`Slow screen load: ${screenName}`),
          { loadTimeMs: loadTime }
        );
      }
    });
  }, [screenName]);
}

// APIリクエスト時間の計測
export function useApiMetrics() {
  const [metrics, setMetrics] = useState({
    requestCount: 0,
    errorCount: 0,
    totalResponseTime: 0
  });
  
  const trackRequest = async (requestPromise: Promise<any>, endpointName: string) => {
    const startTime = Date.now();
    try {
      setMetrics(prev => ({ ...prev, requestCount: prev.requestCount + 1 }));
      const response = await requestPromise;
      const responseTime = Date.now() - startTime;
      
      setMetrics(prev => ({
        ...prev,
        totalResponseTime: prev.totalResponseTime + responseTime
      }));
      
      // 閾値を超えた場合は警告
      if (responseTime > 3000) {
        ErrorTracker.captureException(
          new Error(`Slow API response: ${endpointName}`),
          { responseTimeMs: responseTime }
        );
      }
      
      return response;
    } catch (error) {
      setMetrics(prev => ({ ...prev, errorCount: prev.errorCount + 1 }));
      throw error;
    }
  };
  
  return { metrics, trackRequest };
}
```

## テスト戦略

品質を確保するための包括的なテスト戦略を実装します：

### テスト種類と対象

- **単体テスト** (カバレッジ目標: 75%以上)
  - コンポーネント
  - ユーティリティ関数
  - Redux ストア

- **統合テスト** (カバレッジ目標: 60%以上)
  - 画面フロー
  - API連携
  - 状態管理

- **E2Eテスト**
  - 主要ユーザーフローの自動テスト
  - クリティカルパスの定期的テスト
  - エッジケースの検証

### テスト実装ガイドライン

```tsx
// コンポーネントテストの例
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('should handle input changes', () => {
    const { getByPlaceholderText } = render(<LoginForm onSubmit={jest.fn()} />);
    
    const emailInput = getByPlaceholderText('メールアドレス');
    const passwordInput = getByPlaceholderText('パスワード');
    
    fireEvent.changeText(emailInput, 'test@example.com');
    fireEvent.changeText(passwordInput, 'password123');
    
    expect(emailInput.props.value).toBe('test@example.com');
    expect(passwordInput.props.value).toBe('password123');
  });
  
  it('should validate inputs before submission', () => {
    const mockSubmit = jest.fn();
    const { getByText, getByPlaceholderText } = render(
      <LoginForm onSubmit={mockSubmit} />
    );
    
    // 空の状態で送信
    fireEvent.press(getByText('ログイン'));
    expect(mockSubmit).not.toHaveBeenCalled();
    
    // 有効なデータで送信
    fireEvent.changeText(getByPlaceholderText('メールアドレス'), 'test@example.com');
    fireEvent.changeText(getByPlaceholderText('パスワード'), 'password123');
    fireEvent.press(getByText('ログイン'));
    
    expect(mockSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123'
    });
  });
});
```

## オフライン対応

アプリケーションのオフライン対応を実装するためのガイドライン：

```tsx
// shared/lib/offline/offline-queue.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';
import { ApiService } from '@/shared/lib/api/api.service';

// オフラインキュー
export class OfflineQueue {
  private static QUEUE_KEY = 'offline_queue';
  
  // キューにリクエストを追加
  static async addToQueue(request: {
    url: string;
    method: string;
    data?: any;
    headers?: Record<string, string>;
    id: string;
  }) {
    try {
      // 現在のキューを取得
      const queueJson = await AsyncStorage.getItem(this.QUEUE_KEY);
      const queue = queueJson ? JSON.parse(queueJson) : [];
      
      // キューに追加
      queue.push({
        ...request,
        timestamp: Date.now()
      });
      
      // 保存
      await AsyncStorage.setItem(this.QUEUE_KEY, JSON.stringify(queue));
      console.log('Request added to offline queue', request);
    } catch (error) {
      console.error('Failed to add request to queue', error);
    }
  }
  
  // キューの処理を試行
  static async processQueue() {
    try {
      const queueJson = await AsyncStorage.getItem(this.QUEUE_KEY);
      if (!queueJson) return;
      
      const queue = JSON.parse(queueJson);
      if (queue.length === 0) return;
      
      console.log(`Processing offline queue: ${queue.length} items`);
      
      // 成功したリクエストのID
      const successfulRequestIds: string[] = [];
      
      // 各リクエストを処理
      for (const request of queue) {
        try {
          await ApiService.instance({
            url: request.url,
            method: request.method,
            data: request.data,
            headers: request.headers
          });
          
          successfulRequestIds.push(request.id);
        } catch (error) {
          console.error('Failed to process queued request', error);
          // 一時的なエラーなら再試行、永続的なエラーなら削除も検討
        }
      }
      
      // 成功したリクエストをキューから削除
      if (successfulRequestIds.length > 0) {
        const updatedQueue = queue.filter(
          (req: any) => !successfulRequestIds.includes(req.id)
        );
        await AsyncStorage.setItem(this.QUEUE_KEY, JSON.stringify(updatedQueue));
      }
    } catch (error) {
      console.error('Failed to process offline queue', error);
    }
  }
}

// ネットワーク状態の監視とキュー処理
export function initOfflineSync() {
  // ネットワーク状態の変化を監視
  NetInfo.addEventListener(state => {
    if (state.isConnected) {
      console.log('Network connected, processing offline queue');
      OfflineQueue.processQueue();
    }
  });
} 