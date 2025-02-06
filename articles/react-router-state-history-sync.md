---
title: "React Routerと独自UI状態の同期管理 - ブラウザ履歴との整合性を保つ実装パターン"
emoji: "🔄"
type: "tech"
topics: ["react", "reactrouter", "typescript", "frontend", "spa"]
published: false
---

# React Router と独自 UI 状態の同期管理

## はじめに

SPA では画面遷移を React Router で管理しながら、モーダルやサブ画面の状態を独自に管理するケースが一般的です。しかし、この 2 つの状態管理を個別に実装すると、ブラウザの戻る/進む操作時に不整合が発生しやすくなります。

本記事では、React Router と UI の状態管理を適切に同期させる実装パターンを解説します。

## 想定読者

- React Router の基本的な使い方を理解している
- SPA での画面遷移やモーダル実装の経験がある
- TypeScript で React アプリケーションを開発している

## 課題: 二重管理による不整合

### 典型的な実装の問題点

```typescript
// 🚫 アンチパターン: 状態と履歴を別々に管理
const openUserModal = (userId: string) => {
  // UIの状態だけ更新
  setModalState({ isOpen: true, userId });
  // ※履歴の更新を忘れがち！
};
```

このような実装では:

1. ブラウザバックした時にモーダルが閉じない
2. 履歴が正しく残らずブックマークが機能しない
3. 進む/戻る操作で状態が壊れる

といった問題が発生します。

### 具体的な不整合シナリオ

1. ユーザーがモーダルを開く

   - UI の状態: `{ isOpen: true }`
   - URL/履歴: 変更なし ← 問題点

2. ユーザーがブラウザバックを実行
   - UI の状態: `{ isOpen: true }` のまま ← 問題点
   - URL/履歴: 前のページに戻る

## 解決策: カスタムフックによる状態同期

### 1. UI スタック管理のコンテキスト作成

```typescript:src/contexts/UIStackContext.tsx
import { createContext, useContext, useState } from 'react';

type ScreenState = {
  id: string;
  props: Record<string, unknown>;
};

type UIStackContextType = {
  screens: ScreenState[];
  pushScreen: (id: string, props?: Record<string, unknown>) => void;
  popScreen: () => void;
};

const UIStackContext = createContext<UIStackContextType | null>(null);

export function UIStackProvider({ children }: { children: React.ReactNode }) {
  const [screens, setScreens] = useState<ScreenState[]>([]);

  const pushScreen = (id: string, props = {}) => {
    setScreens(prev => [...prev, { id, props }]);
  };

  const popScreen = () => {
    setScreens(prev => prev.slice(0, -1));
  };

  return (
    <UIStackContext.Provider value={{ screens, pushScreen, popScreen }}>
      {children}
    </UIStackContext.Provider>
  );
}

export const useUIStack = () => {
  const context = useContext(UIStackContext);
  if (!context) {
    throw new Error('useUIStack must be used within UIStackProvider');
  }
  return context;
};
```

### 2. 画面遷移管理のカスタムフック

```typescript:src/hooks/useScreenNavigation.ts
import { useNavigate, useLocation } from 'react-router-dom';
import { useUIStack } from '../contexts/UIStackContext';

export function useScreenNavigation() {
  const navigate = useNavigate();
  const location = useLocation();
  const { pushScreen, popScreen } = useUIStack();

  const push = (screenId: string, props: Record<string, unknown> = {}) => {
    // UIスタックと履歴を同時に更新
    pushScreen(screenId, props);
    navigate(location.pathname, {
      state: {
        screens: [...(location.state?.screens || []), { id: screenId, props }],
      },
    });
  };

  const pop = () => {
    // UIスタックと履歴を同時に更新
    popScreen();
    navigate(-1);
  };

  return { push, pop };
}
```

### 3. ブラウザ履歴同期の実装

```typescript:src/hooks/useHistorySync.ts
import { useEffect } from 'react';
import { useNavigationType, NavigationType, useLocation } from 'react-router-dom';
import { useUIStack } from '../contexts/UIStackContext';

export function useHistorySync() {
  const navigationType = useNavigationType();
  const location = useLocation();
  const { screens, pushScreen, popScreen } = useUIStack();

  useEffect(() => {
    // POP (ブラウザバック/フォワード) の場合
    if (navigationType === NavigationType.Pop) {
      const historyScreens = location.state?.screens || [];

      // 履歴の状態とUIスタックを同期
      if (historyScreens.length < screens.length) {
        popScreen();
      } else if (historyScreens.length > screens.length) {
        const lastScreen = historyScreens[historyScreens.length - 1];
        pushScreen(lastScreen.id, lastScreen.props);
      }
    }
  }, [navigationType, location, screens.length]);
}
```

### 4. 実際の使用例

```typescript:src/components/UserProfile.tsx
import { useScreenNavigation } from '../hooks/useScreenNavigation';

export function UserProfile() {
  const { push, pop } = useScreenNavigation();

  const handleUserClick = (userId: string) => {
    // 状態と履歴が自動で同期される
    push('userDetail', { userId });
  };

  const handleClose = () => {
    // 戻る操作も同期される
    pop();
  };

  return (
    // ...
  );
}
```

## 実装のポイント

1. **単一の責任**: 画面遷移に関する操作は必ず`useScreenNavigation`を通して行う
2. **型安全性**: TypeScript の型システムを活用して、画面 ID や props の型を強制する
3. **テスト容易性**: カスタムフックに切り出すことで、ユニットテストが書きやすい
4. **再利用性**: 同じパターンを他の画面やモーダルでも簡単に適用できる

## まとめ

React Router と UI の状態管理を適切に同期させることで:

- ブラウザの戻る/進む操作との整合性が保たれる
- ブックマークやシェアが正しく機能する
- 実装ミスを防ぎやすい堅牢な設計になる

このパターンを活用することで、より信頼性の高い SPA を実装できます。

## 参考資料

- [React Router 公式ドキュメント](https://reactrouter.com/)
- [React Router v6 データ API](https://reactrouter.com/docs/en/v6/data)
- [React Context 公式ドキュメント](https://react.dev/learn/passing-data-deeply-with-context)

---

