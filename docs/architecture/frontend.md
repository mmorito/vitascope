# フロントエンドアーキテクチャ (FSD)

フロントエンドでは、Feature-Sliced Design (FSD) アーキテクチャを採用し、機能の分離と再利用性を高めます。

## FSD の基本原則

- **レイヤーの明確な分離**: 各レイヤーは特定の責任を持ち、下位レイヤーに依存することができる
- **スライスによる機能分割**: 機能ごとにコードを分割し、各スライスは独立して動作可能
- **独立したエンティティ**: ビジネスエンティティを明確に分離し、プロジェクト全体で再利用
- **共有コードの集約**: 共通ユーティリティと再利用可能なコンポーネントを shared レイヤーに配置

## ディレクトリ構造

```
frontend/
├── src/
│   ├── app/        # アプリケーション初期化、プロバイダー、ルーティング
│   ├── processes/  # クロスフィーチャーのワークフロー、グローバルプロセス
│   ├── screens/    # 画面コンポーネント（Webアプリにおけるpagesに相当）
│   ├── widgets/    # 複合UIブロック、画面の独立した部分
│   ├── features/   # ユーザー操作に関連する機能単位
│   ├── entities/   # ビジネスエンティティ（患者、医療記録など）
│   └── shared/     # 共通ユーティリティ、UIキット、API、型定義など
├── assets/         # 静的アセット（画像、フォント等）
└── tests/          # テスト関連ファイル
```

## レイヤー構造と依存関係

FSD では、以下のレイヤー構造と依存関係に従います：

```
app/ → processes/ → screens/ → widgets/ → features/ → entities/ → shared/
```

各レイヤーは下位のレイヤーにのみ依存でき、同じレイヤー内の他のスライスには依存できません。

## React Native + Expo による実装

Vitascope フロントエンドでは、React Native と Expo を使用してクロスプラットフォームアプリを開発します。

### 推奨ライブラリスタック

| カテゴリ           | 推奨ライブラリ         | 用途                              |
| ------------------ | ---------------------- | --------------------------------- |
| 状態管理           | Redux Toolkit          | グローバル状態管理                |
| クエリ管理         | React Query            | API 通信とデータキャッシュ        |
| スタイリング       | NativeWind             | Tailwind CSS ライクなスタイリング |
| ナビゲーション     | React Navigation       | 画面遷移管理                      |
| フォーム管理       | React Hook Form        | フォーム状態と検証                |
| バリデーション     | Zod                    | 型安全なバリデーション            |
| 国際化             | i18next                | 多言語対応                        |
| ストレージ         | AsyncStorage           | ローカルデータ保存                |
| セキュアストレージ | Expo SecureStore       | 機密情報の安全な保存              |
| グラフ             | Victory Native         | データ可視化                      |
| テスト             | Jest + Testing Library | 単体/コンポーネントテスト         |

### インポート構造

プロジェクトでは、バレルパターン（index.ts によるエクスポート）を使用して内部モジュールをカプセル化します：

```tsx
// features/authentication/index.ts
export { LoginForm } from "./ui/LoginForm";
export { RegistrationForm } from "./ui/RegistrationForm";
export { useAuth } from "./model/useAuth";
export type { AuthFormProps } from "./types";
```

また、TypeScript パスエイリアスを使用して相対パスを避けます：

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@app/*": ["src/app/*"],
      "@screens/*": ["src/screens/*"],
      "@widgets/*": ["src/widgets/*"],
      "@features/*": ["src/features/*"],
      "@entities/*": ["src/entities/*"],
      "@shared/*": ["src/shared/*"]
    }
  }
}
```

### Expo の設定

```json
// app.json
{
  "expo": {
    "name": "Vitascope",
    "slug": "vitascope",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "updates": {
      "fallbackToCacheTimeout": 0
    },
    "assetBundlePatterns": ["**/*"],
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.company.vitascope"
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#FFFFFF"
      },
      "package": "com.company.vitascope"
    },
    "web": {
      "favicon": "./assets/favicon.png"
    },
    "plugins": ["expo-secure-store", "expo-health-connect"]
  }
}
```

## 実装ガイドライン

### コンポーネント設計

```tsx
// features/authentication/ui/LoginForm.tsx
import { useState } from "react";
import { Button, TextInput } from "@/shared/ui";
import { useAuth } from "@/entities/session";

export const LoginForm = () => {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const { login, isLoading } = useAuth();

  const handleSubmit = async () => {
    await login(email, password);
  };

  return (
    <form onSubmit={handleSubmit}>
      <TextInput label="メールアドレス" value={email} onChange={setEmail} />
      <TextInput
        label="パスワード"
        type="password"
        value={password}
        onChange={setPassword}
      />
      <Button loading={isLoading} type="submit">
        ログイン
      </Button>
    </form>
  );
};
```

### 公開 API と内部実装の分離

各スライスは、`index.ts` ファイルを通じて公開 API を明示的に定義します：

```ts
// features/authentication/index.ts
export { LoginForm } from "./ui/LoginForm";
export { RegistrationForm } from "./ui/RegistrationForm";
export type { AuthFormProps } from "./types";

// 内部実装は公開しない
// import { validatePassword } from './lib/validators'; // ❌ 公開しない
```

### モデル（エンティティ）の実装

```ts
// entities/patient/model/types.ts
export interface Patient {
  id: string;
  name: string;
  birthDate: Date;
  gender: "male" | "female" | "other";
  medicalRecordIds: string[];
}

// entities/patient/model/store.ts
import { create } from "zustand";
import { Patient } from "./types";
import { fetchPatient } from "../api";

interface PatientStore {
  currentPatient: Patient | null;
  isLoading: boolean;
  error: Error | null;
  loadPatient: (id: string) => Promise<void>;
}

export const usePatientStore = create<PatientStore>((set) => ({
  currentPatient: null,
  isLoading: false,
  error: null,
  loadPatient: async (id) => {
    set({ isLoading: true, error: null });
    try {
      const patient = await fetchPatient(id);
      set({ currentPatient: patient, isLoading: false });
    } catch (err) {
      set({ error: err as Error, isLoading: false });
    }
  },
}));
```

### React Navigation を使用した画面遷移

```tsx
// app/navigation/AppNavigator.tsx
import { NavigationContainer } from "@react-navigation/native";
import { createStackNavigator } from "@react-navigation/stack";
import { createBottomTabNavigator } from "@react-navigation/bottom-tabs";
import { HomeScreen } from "@/screens/home";
import { ProfileScreen } from "@/screens/profile";
import { LoginScreen } from "@/screens/auth/login";
import { useAuth } from "@/entities/session";

const Stack = createStackNavigator();
const Tab = createBottomTabNavigator();

const MainTabs = () => (
  <Tab.Navigator>
    <Tab.Screen name="Home" component={HomeScreen} />
    <Tab.Screen name="Profile" component={ProfileScreen} />
  </Tab.Navigator>
);

export const AppNavigator = () => {
  const { isAuthenticated } = useAuth();

  return (
    <NavigationContainer>
      <Stack.Navigator>
        {isAuthenticated ? (
          <Stack.Screen
            name="Main"
            component={MainTabs}
            options={{ headerShown: false }}
          />
        ) : (
          <Stack.Screen name="Login" component={LoginScreen} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
};
```

### API 通信とデータフェッチング

```tsx
// features/health-data/api/fetchHealthData.ts
import { useQuery } from "@tanstack/react-query";
import { api } from "@/shared/api";
import { HealthData } from "@/entities/health";

export const fetchHealthData = async (userId: string): Promise<HealthData> => {
  const response = await api.get(`/v1/users/${userId}/health-data`);
  return response.data;
};

export const useHealthData = (userId: string) => {
  return useQuery({
    queryKey: ["healthData", userId],
    queryFn: () => fetchHealthData(userId),
    staleTime: 5 * 60 * 1000, // 5分間キャッシュを有効にする
  });
};

// screens/HealthDashboard.tsx での使用例
export const HealthDashboard = ({ userId }: { userId: string }) => {
  const { data, isLoading, error } = useHealthData(userId);

  if (isLoading) return <LoadingIndicator />;
  if (error) return <ErrorDisplay message={error.message} />;

  return (
    <View>
      <Text>心拍数: {data.heartRate} BPM</Text>
      <Text>歩数: {data.steps} 歩</Text>
      {/* 他のヘルスデータの表示 */}
    </View>
  );
};
```

## コンポーネント命名規則

- スライス名: `feature-name`
- ファイル名: PascalCase（コンポーネント）、camelCase（ユーティリティ）
- コンポーネント名: PascalCase
- フック名: `use` + PascalCase
- イベントハンドラ名: `handle` + イベント名

## 状態管理戦略

- **ローカル状態**: React の `useState` と `useReducer`
- **グローバル状態**: Redux Toolkit または Zustand
- **API 状態**: React Query
- **フォーム状態**: React Hook Form

## 共通コンポーネントライブラリ

UI コンポーネントは `shared/ui` に配置し、すべてのスライスから再利用できるようにします：

```tsx
// shared/ui/Button/Button.tsx
import { ButtonProps } from "./types";
import { styled } from "nativewind";

const StyledButton = styled(View, "px-4 py-2 rounded-md");
const ButtonText = styled(Text, "font-semibold text-white");

export const Button = ({
  children,
  variant = "primary",
  size = "medium",
  loading = false,
  ...props
}: ButtonProps) => {
  const variantStyles = {
    primary: "bg-blue-500",
    secondary: "bg-gray-500",
    success: "bg-green-500",
    danger: "bg-red-500",
  }[variant];

  const sizeStyles = {
    small: "py-1 px-3",
    medium: "py-2 px-4",
    large: "py-3 px-6",
  }[size];

  return (
    <StyledButton className={`${variantStyles} ${sizeStyles}`} {...props}>
      {loading ? (
        <ActivityIndicator color="white" size="small" />
      ) : (
        <ButtonText>{children}</ButtonText>
      )}
    </StyledButton>
  );
};
```
