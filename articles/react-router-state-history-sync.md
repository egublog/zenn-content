---
title: "React Routerと独自UI状態の二重管理における課題と解決策"
emoji: "🐡"
type: "tech"
topics: ["react", "reactrouter", "frontend"]
published: false
---

# React Router と独自 UI 状態の二重管理における課題と解決策

シングルページアプリケーション（SPA）では、React Router を利用して URL を管理しながら、内部的にモーダルやサブ画面といった独自の UI 状態をスタック管理するケースがよくあります。これにより、ユーザー体験を向上させることができますが、状態管理とブラウザ履歴の管理を別々に扱うと、同期ミスが起こりやすくなります。今回は、その課題と実装ミスを防ぐための共通化手法について解説します。

## 1. React Router について

React Router は、React アプリケーションにおけるルーティング（画面遷移）をシンプルに実装できるライブラリです。

- **URL 管理**: ページごとに URL を変えることで、ユーザーがブックマークしたり、ブラウザの戻る・進むボタンを使った際にも意図した画面に遷移させることができます。
- **location.state**: 遷移時に任意の状態（state）を付加することで、URL 上に現れない情報を一緒に渡すことができるため、モーダル表示などの一時的な UI 状態管理に応用できます。

## 2. 独自の UI 状態を持つとは

独自の UI 状態とは、URL 自体には反映させずに、画面上の表示状態をアプリ内部の状態で管理する方法です。

:::message
例: モーダル表示

- ユーザーがプロフィールをクリックすると、URL は変えずにモーダルが開き、ユーザー情報が表示される。
  :::

:::message
例: 内部画面のスタック

- チャットアプリでは、メイン画面の上にチャット詳細の画面を重ねる場合、内部のスタックに「詳細画面」を追加しておき、戻る操作でこのスタックをポップすることで、元の状態に戻す、といった実装が考えられます。
  :::

## 3. ブラウザバック/進むの不一致によるつらみ

内部状態（UI 状態のスタック）と React Router の履歴が別個に管理されていると、以下のような不一致が発生する可能性があります。

### 状態のズレ

:::message alert
例えば、モーダルを開いたときに内部では「モーダル表示中」という状態をセットし、同時に React Router の navigate で履歴にエントリを追加しておく必要があります。ここで、どちらか一方だけ更新してしまうと、ブラウザの戻るボタンを押したときに内部状態だけが変わらず、画面と履歴が不一致になってしまいます。
:::

### ブラウザバック時の予期しない動作

履歴上は前のエントリに戻っているのに、内部状態が更新されずモーダルが閉じなかったり、逆に内部状態は戻っているのに履歴上はそのままになっていたりすると、ユーザーは混乱します。

### プログラム的な操作ミス

:::message alert
状態を操作するたびに、対応する履歴操作（例: navigate, history.push, history.goBack など）を手動で行う必要があると、どちらか一方の操作を忘れるなどの実装ミスが起こりやすくなります。
:::

このような「二重管理」状態の不整合は、開発・保守時に非常に悩ましい問題となります。

## 4. 解決策

共通の関数/カスタムフックで状態変更と履歴変更を一貫して行う
この問題の根本解決策は、状態変更と履歴変更を常にペアで実行する仕組みを作ることです。以下のアプローチが考えられます。

### 4.1. カスタムフックの作成例

たとえば、画面を新たに開くときに、内部の UI スタックにプッシュすると同時に、React Router の履歴にもエントリを追加する関数 usePushScreen を実装します。

```typescript
import { useContext } from "react";
import { useNavigate, useLocation } from "react-router-dom";
import { UIStackContext } from "./UIStackContext";

export function usePushScreen() {
  const navigate = useNavigate();
  const location = useLocation();
  const { pushScreen } = useContext(UIStackContext);

  const push = (screenId, additionalState = {}) => {
    // 1. 内部の UI 状態（スタック）を更新する
    pushScreen(screenId);

    // 2. React Router の履歴にもエントリを追加する
    // URL自体は変えずに、location.stateに UI 状態を含める
    navigate(location.pathname, {
      state: { ...location.state, currentScreen: screenId, ...additionalState },
      // 必要に応じて replace: false を明示
    });
  };

  return push;
}
```

このように、push 操作に対して内部状態と履歴の両方を更新する共通の関数を用意することで、更新漏れを防げます。
同様に、戻る操作についてもカスタムフック usePopScreen を作成するとよいでしょう。

### 4.2. 戻る操作のカスタムフック例

```typescript
import { useContext } from "react";
import { useNavigate } from "react-router-dom";
import { UIStackContext } from "./UIStackContext";

export function usePopScreen() {
  const navigate = useNavigate();
  const { popScreen } = useContext(UIStackContext);

  const pop = () => {
    // 1. 内部の UI 状態を 1 つ戻す
    popScreen();

    // 2. 履歴も1つ戻す
    navigate(-1);
  };

  return pop;
}
```

このように、内部状態の変更と履歴の変更を一つの関数にまとめれば、どこで操作しても常に一貫性が保たれ、ブラウザの戻るボタンと自作の戻るボタンの双方で整合性のある挙動が実現できます。

### 4.3. ブラウザバックのイベントを捕捉する

React Router v6 では、useNavigationType フックなどを用いて、ブラウザバックや進む操作を検知することも可能です。これにより、ユーザーがブラウザの戻るボタンを押した際にも、内部の UI スタックを正しく更新することができます。

```typescript
import { useEffect, useRef } from "react";
import {
  useNavigationType,
  NavigationType,
  useLocation,
} from "react-router-dom";
import { useUIStack } from "./UIStackContext";

export function useBackNavigationSync() {
  const navigationType = useNavigationType();
  const location = useLocation();
  const { popScreen } = useUIStack();
  const initialRender = useRef(true);

  useEffect(() => {
    if (initialRender.current) {
      initialRender.current = false;
      return;
    }

    // POP (ブラウザバック) の場合に内部状態も戻す
    if (navigationType === NavigationType.Pop) {
      popScreen();
    }
  }, [navigationType, location, popScreen]);
}
```

このようにカスタムフックを利用して、ブラウザバック操作時に自動で内部状態を更新する仕組みを導入することで、状態と履歴の不整合を未然に防げます。

## 5. まとめ

以下のポイントを押さえることで、複雑な UI 状態とブラウザ履歴の両立を実現できます：

- **React Router の活用**: URL 上の履歴管理は React Router の `navigate` や `location.state` を使って柔軟に行えます
- **独自 UI 状態の管理**: 内部で保持する UI 状態（モーダルやサブ画面のスタック）を適切に設計し、ユーザー体験を向上させる
- **二重管理の落とし穴**: 状態と履歴を別々に管理すると、ブラウザバックや進むときに不整合が生じ、実装ミスも発生しやすい
- **共通化による同期**: カスタムフックや共通の関数を作成し、状態変更と履歴変更を常にペアで実行することで、同期を取りミスを防ぐ

:::message
このような設計パターンを採用することで、複雑な UI 状態とブラウザ履歴の両立をシンプルに管理でき、ユーザー体験の向上と保守性の高いコードベースが実現できます。
:::
