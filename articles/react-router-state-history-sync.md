---
title: "React Routerと独自UI状態の同期管理 - ブラウザ履歴との整合性を保つ実装パターン"
emoji: "🔄"
type: "tech"
topics: ["react", "reactrouter", "typescript", "frontend", "spa"]
published: false
---

# React Router と独自 UI 状態の同期管理

:::message
この記事は React Router v6 以降を対象としています。
:::

## はじめに

SPA では画面遷移を React Router で管理しながら、モーダルやサブ画面の状態を独自に管理するケースが一般的です。しかし、この 2 つの状態管理を個別に実装すると、ブラウザの戻る/進む操作時に不整合が発生しやすくなります。

本記事では、**React Router と UI の状態管理を適切に同期させる実装パターン**を解説します。この記事を読むことで、以下のことが学べます：

- React Router と UI 状態の同期における一般的な問題点
- ブラウザ履歴と UI 状態を一貫して管理するためのパターン
- TypeScript を活用した型安全な実装方法

## 前提知識

この記事を理解するためには、以下の知識・経験があると理解しやすいでしょう：

- React Router の基本的な使い方（特に useNavigate, useLocation の理解）
- SPA での画面遷移やモーダル実装の経験
- TypeScript で React アプリケーションを開発した経験
- React の Context API と useEffect フックの基本的な理解

## 課題: 二重管理による不整合

### 典型的な実装の問題点

多くの SPA では、UI の状態管理とルーティングが別々に実装されています。これが不整合を引き起こす原因となります。

```typescript
// 🚫 アンチパターン: 状態と履歴を別々に管理
const openUserModal = (userId: string) => {
  // UIの状態だけ更新
  setModalState({ isOpen: true, userId });
  // ※履歴の更新を忘れがち！
};
```

このような実装では、以下のような問題が発生します：

1. ブラウザバックした時にモーダルが閉じない
2. 履歴が正しく残らずブックマークが機能しない
3. 進む/戻る操作で状態が壊れる

### 具体的な不整合シナリオ

以下のシナリオを考えてみましょう：

1. ユーザーがモーダルを開く

   - UI の状態: `{ isOpen: true }`
   - URL/履歴: 変更なし ← **問題点**

2. ユーザーがブラウザバックを実行
   - UI の状態: `{ isOpen: true }` のまま ← **問題点**
   - URL/履歴: 前のページに戻る

![状態不整合の図解](/images/state-history-mismatch.png)
_図 1: ブラウザ履歴と UI 状態の不整合_

## 解決策: カスタムフックによる状態同期

この問題を解決するために、以下の 3 つのコンポーネントを実装します：

1. UI スタック管理のコンテキスト
2. 画面遷移管理のカスタムフック
3. ブラウザ履歴同期のカスタムフック

### 1. UI スタック管理のコンテキスト作成

まず、アプリケーション全体で UI の状態を管理するためのコンテキストを作成します。

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

:::details UIStackContext の詳細解説
このコンテキストは、画面やモーダルの状態をスタック構造で管理します。各画面は `id` と `props` を持ち、新しい画面が追加されるとスタックの一番上に積まれます。画面を閉じる際は、スタックの一番上から取り除かれます。
:::

### 2. 画面遷移管理のカスタムフック

次に、UI の状態とブラウザ履歴を同時に更新するためのカスタムフックを作成します。

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

:::message alert
`navigate` 関数の第二引数に `state` を渡すことで、URL を変更せずに履歴スタックに状態を保存できます。これは React Router v6 の重要な機能です。
:::

### 3. ブラウザ履歴同期の実装

最後に、ブラウザの戻る/進む操作と UI 状態を同期させるためのカスタムフックを作成します。

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

:::details 実装のポイント
このフックは `useNavigationType` を使用して、ブラウザの戻る/進む操作を検出します。操作が検出されると、現在の UI 状態と履歴の状態を比較し、必要に応じて同期を行います。
:::

### 4. 実際の使用例

これらのフックを使用して、実際のコンポーネントを実装してみましょう。

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
    <div>
      <h1>ユーザープロフィール</h1>
      <button onClick={() => handleUserClick('user123')}>
        詳細を表示
      </button>
      {/* ... */}
    </div>
  );
}
```

## 実装のポイント

この実装パターンには、以下のような利点があります：

1. **単一の責任**: 画面遷移に関する操作は必ず`useScreenNavigation`を通して行う
2. **型安全性**: TypeScript の型システムを活用して、画面 ID や props の型を強制する
3. **テスト容易性**: カスタムフックに切り出すことで、ユニットテストが書きやすい
4. **再利用性**: 同じパターンを他の画面やモーダルでも簡単に適用できる

:::message
より堅牢な実装にするためには、画面 ID を列挙型（enum）で定義し、props の型も画面ごとに適切に定義することをお勧めします。
:::

## 応用: より型安全な実装

より型安全な実装を目指す場合は、以下のように拡張できます：

```typescript
// 画面IDを列挙型で定義
enum ScreenId {
  UserDetail = "userDetail",
  Settings = "settings",
  // ...
}

// 画面ごとのpropsの型を定義
type ScreenProps = {
  [ScreenId.UserDetail]: { userId: string };
  [ScreenId.Settings]: { section?: string };
  // ...
};

// 型安全なpush関数
function push<T extends ScreenId>(screenId: T, props: ScreenProps[T]) {
  // ...
}
```

## まとめ

React Router と UI の状態管理を適切に同期させることで、以下のメリットが得られます：

- ブラウザの戻る/進む操作との整合性が保たれる
- ブックマークやシェアが正しく機能する
- 実装ミスを防ぎやすい堅牢な設計になる

このパターンを活用することで、より信頼性の高い SPA を実装できます。特に複雑な UI 状態を持つアプリケーションでは、このような同期管理が重要になります。

## 参考資料

- [React Router 公式ドキュメント](https://reactrouter.com/)
- [React Router v6 データ API](https://reactrouter.com/docs/en/v6/data)
- [React Context 公式ドキュメント](https://react.dev/learn/passing-data-deeply-with-context)
- [TypeScript Handbook - Generic Types](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [SPA のルーティングとブラウザ履歴の仕組み](https://developer.mozilla.org/ja/docs/Web/API/History_API)

---
