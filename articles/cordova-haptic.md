---
title: "Cordovaで触覚フィードバック（Haptic機能）を実装する方法"
emoji: "📱"
type: "tech"
topics: ["cordova", "haptic", "javascript", "plugin", "mobile"]
published: false
---

# Cordova アプリに触覚フィードバックを実装する完全ガイド

スマートフォンアプリで「ブルッ」という振動フィードバックを感じたことはありませんか？この触覚フィードバック（Haptic Feedback）は、ユーザー体験を大きく向上させる重要な要素です。

Cordova アプリでもネイティブ同様の Haptic 機能を実装することが可能です。本記事では、実装に必要なプラグインの選定から基本的な実装手順、さらに振動パターンのカスタマイズやベストプラクティスまで紹介します。

:::message
**この記事で学べること：**

- Cordova での Haptic 機能実装の基礎
- 適切なプラグインの選び方
- プラットフォーム別の振動パターンのカスタマイズ方法
- 実装時の注意点とベストプラクティス
  :::

## プラグインの選定

Cordova で Haptic 機能を実装する場合、主に 2 つの選択肢があります：

### 1. 公式プラグイン: cordova-plugin-vibration

シンプルな振動機能が必要な場合におすすめです。

| メリット・デメリット | 詳細                                                           |
| -------------------- | -------------------------------------------------------------- |
| ✅                   | Cordova 公式の振動プラグインで、`navigator.vibrate` API を提供 |
| ✅                   | 基本的な振動機能を実装可能                                     |
| ⚠️                   | iOS では時間指定や振動パターンに制限あり                       |

:::message alert
**iOS の制限事項**：iOS は時間指定を無視して固定長でしか振動しません。また、振動パターン（複数回のオン・オフ）も iOS 側ではサポート外です。
:::

### 2. 高度な Haptic フィードバック対応プラグイン: cordova-plugin-haptic

より細かな制御が必要な場合におすすめです。

| メリット | 詳細                                                                      |
| -------- | ------------------------------------------------------------------------- |
| ✅       | Android と iOS 両方の**ネイティブの触覚フィードバックパターン**を利用可能 |
| ✅       | プラットフォームごとの実装を意識する必要が少ない                          |
| ✅       | OS が推奨する標準の触覚体験を実現                                         |

## 基本実装

### プラグインのインストール

まずは、プラグインをインストールします：

```bash
# プラグインのインストール
cordova plugin add cordova-plugin-haptic
```

### 基本的な使い方

プラグインは以下のようなシンプルな API を提供します：

```javascript
cordova.plugins.hapticPlugin.sendHapticFeedback(
  androidType, // Androidデバイス用の振動タイプ
  iosType, // iOSデバイス用の振動タイプ
  errorCallback // エラー発生時のコールバック
);
```

::::details 実装例とデモコード

```javascript
// デバイス準備完了時に実行
document.addEventListener("deviceready", () => {
  // 成功時の触覚フィードバック関数
  function playSuccessHaptic() {
    cordova.plugins.hapticPlugin.sendHapticFeedback(
      "CONFIRM", // Android用: 成功・確定操作の振動
      "Success", // iOS用: 成功通知の振動
      (err) => {
        if (err) console.warn("Haptic Error:", err);
      }
    );
  }

  // ボタンにイベントリスナーを追加
  document
    .getElementById("successBtn")
    .addEventListener("click", playSuccessHaptic);
});
```

:::message
上記のコードは、ボタンクリック時に成功を示す触覚フィードバックを再生します。
:::
::::

## 振動パターンのカスタマイズ

### Android のフィードバックタイプ

Android では以下のような標準パターンが用意されています：

| パターン           | 用途       | 特徴                      |
| ------------------ | ---------- | ------------------------- |
| `VIRTUAL_KEY`      | ボタン押下 | 短く軽い振動              |
| `LONG_PRESS`       | 長押し     | やや長めの振動            |
| `KEYBOARD_TAP`     | キーボード | 極めて短い振動            |
| `CONFIRM / REJECT` | 成功/失敗  | Android 12 以降で利用可能 |

### iOS のフィードバックタイプ

iOS では 3 種類のフィードバックカテゴリが用意されています：

::::details iOS 振動パターンの詳細

#### 1. Impact 系

物体同士が衝突するような感覚を表現

- `ImpactLight` / `ImpactMedium` / `ImpactHeavy`
- `ImpactSoft` / `ImpactRigid`

#### 2. Notification 系

通知やアラートに適した振動

- `Success` / `Warning` / `Error`

#### 3. Selection 系

- `SelectionChanged`: ピッカーなどの選択変更時

:::message
各パターンは特定のユーザーアクションに最適化されています。例えば、`Success`は操作成功時、`Error`はエラー発生時に使用するのが適切です。
:::
::::

## ベストプラクティス

### 1. システム標準パターンの適切な使用

:::message
OS のガイドラインに沿ったフィードバック種類を選び、一貫した体験を提供しましょう。
例えば、成功時に Error パターンを使用するのは避けましょう。
:::

### 2. 頻度と強さの調整

振動の使用頻度と強さは、ユーザー体験に大きく影響します：

| 操作タイプ   | 推奨される振動 | 注意点               |
| ------------ | -------------- | -------------------- |
| 頻繁な操作   | 短く弱い振動   | キーボード入力など   |
| 重要な操作   | やや強い振動   | 送信完了、エラーなど |
| 連続的な操作 | 振動を控えめに | スクロールなど       |

### 3. 一貫性の維持

:::details 一貫性を保つためのポイント

- ✅ 同じ操作には同じ Haptic を使用
- ✅ プラットフォームの慣習に合わせる
- ❌ 画面ごとに異なる振動パターンを使用しない

**悪い例**：

- ログイン画面では成功時に強い振動、商品追加時には弱い振動を使用
- Android と iOS で全く異なる振動パターンを使用

**良い例**：

- すべての確認ダイアログで同じ振動パターン
- 各プラットフォームの標準的な振動感覚に合わせる
  :::

### 4. 視覚・聴覚効果との調和

- ✅ アニメーションやサウンドと同期
- ✅ フィードバックの意味を明確に
- ❌ 触覚単独での使用は避ける

### 5. ユーザー設定への配慮

:::message
システムの振動設定を OFF にしているユーザーへの配慮として、アプリ内で ON/OFF 切り替えを提供することを検討しましょう。
:::

## 実装例：様々なユースケース

::::details ユースケース別実装例

### ボタン押下時の振動

```javascript
// ボタン押下時の軽い振動
function buttonTapFeedback() {
  cordova.plugins.hapticPlugin.sendHapticFeedback(
    "VIRTUAL_KEY", // Android
    "ImpactLight", // iOS
    console.error
  );
}
```

### フォーム送信成功時

```javascript
// フォーム送信成功時の振動
function formSubmitSuccessFeedback() {
  cordova.plugins.hapticPlugin.sendHapticFeedback(
    "CONFIRM", // Android
    "Success", // iOS
    console.error
  );
}
```

### エラー発生時

```javascript
// エラー発生時の振動
function errorFeedback() {
  cordova.plugins.hapticPlugin.sendHapticFeedback(
    "REJECT", // Android
    "Error", // iOS
    console.error
  );
}
```

::::

## おわりに

Haptic 機能は画面上の操作を物理的な感覚へと拡張する強力な手段です。適切に導入することで、アプリの UX を大きく向上させることができます。

:::message
**実装時のポイント：**

1. ユースケースに合わせて適切なプラグインを選択
2. プラットフォーム固有の振動パターンを理解
3. ユーザー体験を考慮した適切な使用
4. システム設定への配慮
   :::

## 参考リンク

@[card](https://cordova.apache.org/)
@[card](https://developer.apple.com/design/human-interface-guidelines/)
@[card](https://developer.android.com/guide/topics/ui/haptic-feedback)

<!-- 以下はGitHubリポジトリへのリンク -->

https://github.com/example/cordova-plugin-haptic
