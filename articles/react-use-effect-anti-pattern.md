---
title: "ReactのuseEffect徹底解説：初級者が陥りやすい誤解とアンチパターン"
emoji: "🙆"
type: "tech"
topics: ["react", "hooks", "useEffect"]
published: false
---

React Hooks の中でも特に強力で便利な useEffect フックですが、その挙動を正しく理解しないとバグや非効率なコードの原因になります。本記事では、useEffect の基本から、初級者でもハマりがちな誤解・アンチパターンを網羅し、実際の開発現場での試行錯誤を交えて深掘りします。最後に、ベストプラクティスや他の Hook との組み合わせによる最適な活用方法も紹介します。

## useEffect の基本的な動作のおさらい

まずは useEffect の挙動を再確認しましょう。useEffect は**副作用（side effect）**を扱うための Hook で、レンダーの結果を DOM に反映した後に実行されます。副作用とは、データの取得やログの出力、ブラウザ API の操作など、純粋なレンダリング以外の処理を指します。

### 実行タイミング

コンポーネントの初回レンダー後、および依存している値が更新された後に実行されます。クラスコンポーネントに例えると、componentDidMount と componentDidUpdate を統合したような動作ですが、厳密には「レンダーのたびに実行される」ものだと考えると分かりやすいでしょう。

:::message
React は描画後に useEffect 内の関数（エフェクト）を実行し、画面更新と副作用の同期を取ります。デフォルトでは依存配列を指定しない限り毎回のレンダー後に実行される点に注意が必要です。
:::

### 依存配列 (Dependency Array)

useEffect の第二引数に配列で渡す値のリストです。この配列に含まれる値が変化したときだけエフェクトを再実行するよう制御できます。

- `useEffect(() => {...}, [foo])` - foo が変わった時にのみ実行
- `useEffect(() => {...}, [])` - 初回マウント時のみ実行
- `useEffect(() => {...})` - 毎レンダーで実行

:::message alert
React 18 の Strict Mode 環境下では副作用のバグ検出のため、開発時に限り初回エフェクトが二重に実行されます（マウント → アンマウント → 再マウント）。これは意図的な挙動で、正しくクリーンアップ処理が書けているかをテストするためです。
:::

## useEffect のよくある誤解・アンチパターン

useEffect は強力な反面、その使い方を誤るとバグや非効率を招きます。以下では、現場で実際に遭遇しがちなアンチパターンとその誤解をすべて網羅し、それぞれについて具体例を示しながら解説します。

### 1. 不要な useEffect の使用

誤解: 「とりあえず何か処理が必要なら useEffect に入れておけばよい」という考え方です。

アンチパターン例:

```jsx
interface Props {
  value: number;
}

function MyComponent({ value }: Props) {
  const [doubled, setDoubled] = useState<number>(0);
  // アンチパターン: 単なる計算結果をuseEffectで状態管理
  useEffect(() => {
    setDoubled(value * 2);
  }, [value]);
  // ...
}
```

:::message
このコードでは props.value の変化に対して副作用として setDoubled を呼んでいますが、実際には副作用ではなく単なる演算です。
:::

改善例:

```jsx
interface Props {
  value: number;
}

function MyComponent({ value }: Props) {
  // 単純に計算するだけで良い（必要ならuseMemoでパフォーマンス最適化）
  const doubled = useMemo(() => value * 2, [value]);
  // ...
}
```

### 2. 依存配列の誤用

誤解: 依存配列の役割を誤解すると、エフェクトの再実行タイミングを間違えたり、逆に実行されるべきタイミングでされなかったりします。

アンチパターン例:

```jsx
// 誤った例：value変化時に動作させたいのに依存配列を空にしている
useEffect(() => {
  console.log("Value changed:", props.value);
}, []); // これだと初回しか実行されない！
```

改善例:

```jsx
interface Filters {
  status: string;
  category: string[];
  date: Date;
}

function Component({ filters }: { filters: Filters }) {
  // 注意: JSON.stringifyはパフォーマンスに影響を与える可能性があります
  // 頻繁な更新が予想される場合は、個々のフィールドを依存配列に指定することを検討
  const stableFilters = useMemo(() => filters, [JSON.stringify(filters)]);
  
  // より良い方法：
  const { status, category, date } = filters;
  const stableFiltersOptimized = useMemo(
    () => ({ status, category, date }),
    [status, JSON.stringify(category), date.toISOString()]
  );

  useEffect(() => {
    setCount((prev) => prev + 1);
  }, [stableFiltersOptimized]);
}
```

### 3. 非同期処理の誤った扱い

誤解: useEffect 内で直接 async/await を使っても問題ないと思いがちです。

アンチパターン例:

```jsx
useEffect(async () => {
  const res = await fetchData();
  setData(res);
}, []);
```

:::message alert
上記のコードは「不正な戻り値（クリーンアップ関数ではない）を返した」とみなされ、警告が出ます。
:::

正しい例:

```jsx
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

interface UserData {
  id: number;
  name: string;
  email: string;
}

function UserProfile() {
  const [data, setData] = useState<UserData | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState<boolean>(false);

  useEffect(() => {
    let isMounted = true;
    
    const fetchData = async () => {
      setIsLoading(true);
      try {
        const response = await fetch('/api/user');
        const result: ApiResponse<UserData> = await response.json();
        
        if (!response.ok) {
          throw new Error(result.message);
        }
        
        if (isMounted) {
          setData(result.data);
        }
      } catch (e) {
        if (isMounted) {
          setError(e instanceof Error ? e : new Error('An error occurred'));
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
  }, []);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!data) return null;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}
```

:::message
非同期処理を行う場合は、エフェクトの中で async 関数を定義してそれを呼び出くか、Promise.then チェーンを使うことをお勧めします。
:::

## まとめ

本記事では、useEffect の基本的な使い方から、よくある誤解やアンチパターンまで詳しく解説しました。重要なポイントを振り返ってみましょう：

### useEffect の基本

- useEffect はレンダー後に実行される副作用を扱うための Hook
- 依存配列を使って実行タイミングを制御可能
- React 18 の Strict Mode では開発時に 2 回実行される

### 避けるべきアンチパターン

1. 単純な計算や状態更新に不必要に useEffect を使用
2. 依存配列の誤った指定や省略
3. 非同期処理の不適切な実装

### ベストプラクティス

- 本当に副作用が必要な場合のみ useEffect を使用
- 依存配列は必要最小限に保つ
- 非同期処理では適切なクリーンアップ処理を実装
- 可能な限り他の Hooks（useMemo, useCallback など）の使用を検討
