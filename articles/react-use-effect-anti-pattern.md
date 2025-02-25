---
title: "ReactのuseEffect：よくある誤解とアンチパターン"
emoji: "🎯"
type: "tech"
topics: ["react", "typescript", "hooks", "frontend"]
published: false
---

## はじめに

React 開発で避けて通れない`useEffect`。シンプルに見えて奥が深く、使い方を間違えると思わぬバグの温床になります。本記事では、現場で実際によく遭遇する誤用パターンと、その正しい解決方法を具体的なコード例とともに解説します。

:::message
この記事は、React 中級者を目指すフロントエンドエンジニアを対象としています。React の基本的な概念は理解している前提で解説を進めます。
:::

## 前提条件

この記事を読むにあたって、以下の知識があると理解しやすいでしょう：

- React の基本的な概念（コンポーネント、props、state）
- 関数コンポーネントの基礎
- React Hooks の基本的な使い方

使用する環境：

- React 18.x
- TypeScript 5.x

## useEffect の基本をおさらい

### そもそも useEffect とは？

`useEffect`は**副作用（side effect）** を扱うための Hook です。

- 副作用の例：
  - API からのデータ取得
  - DOM の直接操作
  - タイマーのセット
  - イベントリスナーの登録/解除

### 実行タイミングを理解する

```tsx:基本的な構文
useEffect(
  () => {
    // 実行したい副作用
    return () => {
      // クリーンアップ処理（必要な場合）
    };
  },
  [
    /* 依存配列 */
  ]
);
```

:::message alert
React 18 の Strict Mode では、開発時に副作用が 2 回実行されます！
これは意図的な挙動で、クリーンアップ処理の漏れを発見するためです。
:::

## よくある誤解・アンチパターン

### 1. 不要な useEffect の使用 🚫

#### 悪い例：単純な計算を useEffect で行う

```tsx:アンチパターン
function PriceDisplay({ basePrice }: { basePrice: number }) {
  const [total, setTotal] = useState(0);

  // 🚫 アンチパターン
  useEffect(() => {
    setTotal(basePrice * 1.1); // 消費税計算
  }, [basePrice]);

  return <div>税込価格: {total}円</div>;
}
```

#### 良い例：直接計算するか、useMemo を使う

```tsx:推奨パターン
function PriceDisplay({ basePrice }: { basePrice: number }) {
  // ✅ シンプルな計算なら直接行う
  const total = basePrice * 1.1;

  // 🆗 計算コストが高い場合はuseMemoを検討
  // const total = useMemo(() => {
  //   return complexTaxCalculation(basePrice);
  // }, [basePrice]);

  return <div>税込価格: {total}円</div>;
}
```

:::message
useEffect は「副作用」のために使うものです。単純な計算や状態更新には使わないようにしましょう。
:::

### 2. 依存配列の誤用 🚫

#### 悪い例：必要な依存関係の漏れ

```tsx:アンチパターン
function SearchComponent({ query, filters }) {
  // 🚫 filtersの変更を検知できない
  useEffect(() => {
    searchAPI(query, filters);
  }, [query]); // filtersが依存配列にない！
}
```

#### 良い例：必要な依存関係をすべて含める

```tsx:推奨パターン
function SearchComponent({ query, filters }) {
  // ✅ すべての依存関係を含める
  useEffect(() => {
    searchAPI(query, filters);
  }, [query, JSON.stringify(filters)]); // オブジェクトは安定した参照に変換
}
```

### 3. 非同期処理の誤った実装 🚫

#### 悪い例：直接 async 関数を渡す

```tsx:アンチパターン
// 🚫 アンチパターン
useEffect(async () => {
  const data = await fetchUserData(userId);
  setUserData(data);
}, [userId]);
```

#### 良い例：内部で非同期関数を定義

```tsx:推奨パターン
function UserProfile({ userId }: { userId: string }) {
  const [userData, setUserData] = useState<UserData | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let isMounted = true;

    const fetchData = async () => {
      setIsLoading(true);
      try {
        const data = await fetchUserData(userId);
        if (isMounted) {
          setUserData(data);
        }
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err : new Error("Fetch failed"));
        }
      } finally {
        if (isMounted) {
          setIsLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isMounted = false;
    };
  }, [userId]);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!userData) return null;

  return (
    <div>
      <h2>{userData.name}</h2>
      <p>{userData.email}</p>
    </div>
  );
}
```

:::details useEffect での非同期処理のベストプラクティス

1. 非同期関数は必ず内部で定義する
2. マウント状態を追跡する変数を用意する
3. クリーンアップ関数でマウント状態を更新する
4. 状態更新前に必ずマウント状態をチェックする
   :::

:::message
非同期処理を含む useEffect では、必ずクリーンアップ処理を実装しましょう。
コンポーネントがアンマウントされた後に状態更新が走るのを防ぐことができます。
:::

## useEffect の代替手段

場合によっては、useEffect を使わずに問題を解決できることがあります。

### カスタムフックの活用

```tsx:カスタムフック例
function useUserData(userId: string) {
  const [userData, setUserData] = useState<UserData | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    // 上記の実装と同じ
  }, [userId]);

  return { userData, isLoading, error };
}

// 使用例
function UserProfile({ userId }: { userId: string }) {
  const { userData, isLoading, error } = useUserData(userId);

  // 以下、表示ロジック
}
```

### React Query などのライブラリの活用

```tsx:React Query例
import { useQuery } from 'react-query';

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error } = useQuery(
    ['user', userId],
    () => fetchUserData(userId)
  );

  // 以下、表示ロジック
}
```

## まとめ

useEffect の正しい使い方のポイントは 3 つです：

1. 本当に副作用が必要な場合のみ使用する
2. 依存配列は必要最小限かつ漏れなく設定する
3. 非同期処理では適切なクリーンアップを忘れずに

:::message
useEffect に頼る前に、他の Hooks（useMemo, useCallback など）や専用ライブラリで解決できないか検討することをお勧めします。
:::

## 参考資料

https://react.dev/reference/react/useEffect

https://overreacted.io/ja/a-complete-guide-to-useeffect/

https://kentcdodds.com/blog/useeffect-vs-uselayouteffect
