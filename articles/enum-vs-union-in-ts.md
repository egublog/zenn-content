---
title: "TypeScript における enum と as const の比較：なぜ enum を避けるべきか"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "enum", "frontend", "javascript"]
published: false
---

# TypeScript における enum と as const の比較：なぜ enum を避けるべきか

こんにちは！今回は TypeScript における enum と`as const`を使ったユニオン型の違いについて、そしてなぜ多くの開発者が enum の使用を避けるようになってきているのかを具体例を交えて解説します。

:::message
**この記事のポイント**

- TypeScript における enum と`as const`の違いを理解する
- それぞれのアプローチのメリット・デメリットを把握する
- 実務でどちらを選ぶべきかの判断基準を学ぶ
  :::

## はじめに

TypeScript の型システムを活用するとき、特定の値の集合を表現するために、enum と`as const`を使ったユニオン型という 2 つの主要なアプローチがあります。この記事では、これらの違いを理解し、それぞれのメリット・デメリットを把握することで、より良いコードを書くための知識を身につけましょう。

## 前提条件

この記事を理解するためには、以下の知識があると役立ちます：

- JavaScript の基本的な知識
- TypeScript の基本的な型システムの理解
- モジュールやバンドラーの基本概念

使用する環境：

- TypeScript 4.0 以上
- 任意のコードエディタ（VS Code 推奨）

## enum とは

enum は「列挙型」と呼ばれるもので、名前付きの定数セットを定義する方法です。

```typescript
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

// 使用例
function move(direction: Direction) {
  switch (direction) {
    case Direction.Up:
      return { x: 0, y: 1 };
    case Direction.Down:
      return { x: 0, y: -1 };
    case Direction.Left:
      return { x: -1, y: 0 };
    case Direction.Right:
      return { x: 1, y: 0 };
  }
}

const result = move(Direction.Up); // { x: 0, y: 1 }
```

## as const とユニオン型

一方、`as const`はオブジェクトやリテラル値に対して、それらを**読み取り専用かつ具体的なリテラル型**として扱うよう指示するものです。これを組み合わせることで、enum と似た機能を実現できます。

```typescript
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

// 型の抽出
type Direction = (typeof Direction)[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT" という型になる

// 使用例
function move(direction: Direction) {
  switch (direction) {
    case Direction.Up:
      return { x: 0, y: 1 };
    case Direction.Down:
      return { x: 0, y: -1 };
    case Direction.Left:
      return { x: -1, y: 0 };
    case Direction.Right:
      return { x: 1, y: 0 };
  }
}

const result = move(Direction.Up); // { x: 0, y: 1 }
```

## なぜ enum より as const が推奨されるのか

enum が「良くない」とされる理由はいくつかあります。具体的な例を挙げながら解説します。

:::details 目次

1. バンドルサイズの問題
2. 型の安全性
3. Tree-shaking との相性
4. 拡張性とパターンマッチング
5. 実際のコード例：実務での利用ケース
   :::

### 1. バンドルサイズの問題

enum は実行時にも JavaScript オブジェクトとして存在しますが、`as const`を使った場合はコンパイル時に型情報のみが使われ、実行時のコードは最小限になります。

::::details コンパイル結果の比較

#### enum の場合（コンパイル後）

```javascript
// コンパイル後のJavaScript
var Direction;
(function (Direction) {
  Direction["Up"] = "UP";
  Direction["Down"] = "DOWN";
  Direction["Left"] = "LEFT";
  Direction["Right"] = "RIGHT";
})(Direction || (Direction = {}));
```

#### as const の場合（コンパイル後）

```javascript
// コンパイル後のJavaScript
var Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
};
```

numeric な enum の場合は特に冗長なコードが生成されます：

```typescript
// TypeScript
enum NumericDirection {
  Up, // 0
  Down, // 1
  Left, // 2
  Right, // 3
}
```

```javascript
// コンパイル後のJavaScript
var NumericDirection;
(function (NumericDirection) {
  NumericDirection[(NumericDirection["Up"] = 0)] = "Up";
  NumericDirection[(NumericDirection["Down"] = 1)] = "Down";
  NumericDirection[(NumericDirection["Left"] = 2)] = "Left";
  NumericDirection[(NumericDirection["Right"] = 3)] = "Right";
})(NumericDirection || (NumericDirection = {}));
```

::::

:::message
この違いは小さなプロジェクトでは問題ないかもしれませんが、大規模なアプリケーションでは積み重なってバンドルサイズに影響します。
:::

### 2. 型の安全性

enum には意外な落とし穴があります。数値を使った enum の場合、任意の数値を代入できてしまう問題があります。

::::details 型安全性の比較

```typescript
enum NumericDirection {
  Up, // 0
  Down, // 1
  Left, // 2
  Right, // 3
}

function move(direction: NumericDirection) {
  // ...
}

move(999); // コンパイルエラーにならない！
```

一方、`as const`とユニオン型を使った場合は厳格に型チェックが行われます：

```typescript
const NumericDirection = {
  Up: 0,
  Down: 1,
  Left: 2,
  Right: 3,
} as const;

type NumericDirection =
  (typeof NumericDirection)[keyof typeof NumericDirection];
// 0 | 1 | 2 | 3

function move(direction: NumericDirection) {
  // ...
}

move(999); // 型エラー: '999' は型 '0 | 1 | 2 | 3' に代入できません
```

::::

:::message alert
数値 enum を使用すると、定義されていない値でも型チェックをすり抜けてしまう危険性があります。これは予期せぬバグの原因になりえます。
:::

### 3. Tree-shaking との相性

モダンな JavaScript のバンドラは「**tree-shaking**」という機能を使って、使用されていないコードを除去します。`as const`はこのプロセスと相性が良いですが、enum はそうではありません。

| アプローチ | Tree-shaking との相性 | 理由                             |
| ---------- | --------------------- | -------------------------------- |
| enum       | ❌ 良くない           | 実行時にオブジェクト全体が必要   |
| as const   | ✅ 良い               | 使用する値だけが残る可能性がある |

::::details Tree-shaking の例

```typescript
// ファイル: constants.ts
export enum BigEnum {
  Value1 = "value1",
  Value2 = "value2",
  // ... 100個以上の値
}

// ファイル: app.ts
import { BigEnum } from "./constants";

// Value1だけを使っているが、BigEnum全体がバンドルに含まれる
console.log(BigEnum.Value1);
```

`as const`を使った場合：

```typescript
// ファイル: constants.ts
export const BigConst = {
  Value1: "value1",
  Value2: "value2",
  // ... 100個以上の値
} as const;

export type BigConstType = (typeof BigConst)[keyof typeof BigConst];

// ファイル: app.ts
import { BigConst } from "./constants";

// Value1だけを使う場合、バンドラはValue1だけを含めることが可能
console.log(BigConst.Value1);
```

::::

### 4. 拡張性とパターンマッチング

`as const`を使ったアプローチでは、型定義と値の分離が可能です。これにより、型のユーティリティを使った高度な型操作が容易になります。

::::details 高度な型操作の例

```typescript
const ColorCodes = {
  Red: "#FF0000",
  Green: "#00FF00",
  Blue: "#0000FF",
} as const;

// 値の型の抽出
type ColorCode = (typeof ColorCodes)[keyof typeof ColorCodes];
// "#FF0000" | "#00FF00" | "#0000FF"

// キーの型の抽出
type ColorName = keyof typeof ColorCodes;
// "Red" | "Green" | "Blue"

// マッピング関数
function getColorName(code: ColorCode): ColorName {
  const entries = Object.entries(ColorCodes) as [ColorName, ColorCode][];
  const found = entries.find(([_, value]) => value === code);
  if (!found) throw new Error(`Invalid color code: ${code}`);
  return found[0];
}

console.log(getColorName("#FF0000")); // "Red"
```

::::

:::message
このような型操作は、enum では難しいか、より冗長になります。`as const`を使うと型システムの柔軟性を最大限に活用できます。
:::

### 5. 実際のコード例：実務での利用ケース

実務では以下のような場面で明確な違いが出てきます。例えば、API からのレスポンスを型安全に扱う場合：

::::details 実務での利用ケース比較

#### enum を使った場合

```typescript
// enumを使った場合
enum UserRole {
  Admin = "ADMIN",
  User = "USER",
  Guest = "GUEST",
}

type User = {
  id: string;
  name: string;
  role: UserRole;
};

function hasAdminAccess(user: User): boolean {
  return user.role === UserRole.Admin;
}

// APIからのレスポンス
const apiResponse = {
  id: "123",
  name: "山田太郎",
  role: "ADMIN",
};

// エラー！string型は直接UserRole型に割り当てられない
const user: User = apiResponse;

// 正しくは：
const user: User = {
  ...apiResponse,
  role: apiResponse.role as UserRole, // 型アサーション
};
```

#### as const を使った場合

```typescript
const UserRole = {
  Admin: "ADMIN",
  User: "USER",
  Guest: "GUEST",
} as const;

type UserRole = (typeof UserRole)[keyof typeof UserRole];

type User = {
  id: string;
  name: string;
  role: UserRole;
};

function hasAdminAccess(user: User): boolean {
  return user.role === UserRole.Admin;
}

// APIからのレスポンス
const apiResponse = {
  id: "123",
  name: "山田太郎",
  role: "ADMIN",
};

// 型推論を活用できる
const user: User = {
  ...apiResponse,
  // roleの値が "ADMIN" であれば、それは自動的にUserRole型として認識される
  role: apiResponse.role as UserRole,
};
```

::::

## まとめ：どちらを使うべきか

TypeScript における定数セットの定義方法として、enum と`as const`を比較してきました。

### as const アプローチのメリット

| メリット             | 説明                              |
| -------------------- | --------------------------------- |
| 小さなバンドルサイズ | 実行時のコードが最小限            |
| 厳格な型チェック     | リテラル型による厳密な型安全性    |
| Tree-shaking 対応    | 未使用コードの削除が容易          |
| 柔軟な型操作         | TypeScript の型システムを最大活用 |
| シンプルなコード生成 | 読みやすい JavaScript コード      |

### enum が有用な場面

:::details enum が有用な場面

1. 言語レベルでのサポートが明示的
2. IDE での自動補完が直感的
3. 値とその名前を両方向で参照したい場合（特に数値 enum）

```typescript
// 数値enumの両方向参照
enum Direction {
  Up, // 0
  Down, // 1
  Left, // 2
  Right, // 3
}

// 名前から値
const value = Direction.Up; // 0

// 値から名前
const name = Direction[0]; // "Up"
```

:::

:::message
最終的には、プロジェクトの要件やチームの好みに応じて選択すべきですが、TypeScript コミュニティ全体としては`as const`アプローチが推奨される傾向にあります。特にバンドルサイズや型の安全性を重視する大規模プロジェクトでは、`as const`を使うことでより堅牢なコードベースを構築できるでしょう。
:::

どちらを選ぶにしても、一貫性を持って使うことが重要です。プロジェクト内で混在させると、コードの可読性や保守性が低下する可能性があります。

## 参考資料

@[card](https://www.typescriptlang.org/docs/handbook/enums.html)
@[card](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#const-assertions)
@[card](https://engineering.linecorp.com/ja/blog/typescript-enum-tree-shaking)
@[card](https://github.com/microsoft/TypeScript/wiki/Performance#preferring-interfaces-over-intersections)
