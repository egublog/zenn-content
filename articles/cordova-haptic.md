---
title: "Cordovaで触覚フィードバック（Haptic機能）を実装する方法"
emoji: "📱"
type: "tech"
topics: ["cordova", "haptic", "javascript", "plugin", "mobile"]
published: false
---

Cordova アプリでもネイティブ同様の Haptic 機能を実装することが可能です。本記事では、Cordova で Haptic フィードバックを実装する際に役立つプラグイン選定から基本的な実装手順、さらに振動パターンのカスタマイズやベストプラクティスを紹介します。

## プラグインの選定

### 公式プラグイン: cordova-plugin-vibration

- Cordova 公式の振動プラグインで、`navigator.vibrate` API を提供
- 基本的な振動機能を実装可能
- iOS では時間指定や振動パターンに制限あり

:::message alert
iOS は時間指定を無視して固定長でしか振動しません。また、振動パターン（複数回のオン・オフ）も iOS 側ではサポート外です。
:::

### 高度な Haptic フィードバック対応プラグイン: cordova-plugin-haptic

- Android と iOS 両方の**ネイティブの触覚フィードバックパターン**を利用可能
- プラットフォームごとの実装を意識する必要が少ない
- OS が推奨する標準の触覚体験を実現

## 基本実装

### プラグインのインストール

```bash
cordova plugin add cordova-plugin-haptic
```

### 基本的な使い方

```javascript
cordova.plugins.hapticPlugin.sendHapticFeedback(
  androidType,
  iosType,
  errorCallback
);
```

実装例:

```javascript
document.addEventListener("deviceready", () => {
  function playSuccessHaptic() {
    cordova.plugins.hapticPlugin.sendHapticFeedback(
      "CONFIRM", // Android用: 成功・確定操作の振動
      "Success", // iOS用: 成功通知の振動
      (err) => {
        if (err) console.warn("Haptic Error:", err);
      }
    );
  }

  document
    .getElementById("successBtn")
    .addEventListener("click", playSuccessHaptic);
});
```

## 振動パターンのカスタマイズ

### Android のフィードバックタイプ

- `VIRTUAL_KEY`: 画面上のボタンやソフトキー押下
- `LONG_PRESS`: 長押し
- `KEYBOARD_TAP`: キーボードタップ
- `CONFIRM / REJECT`: 成功/失敗動作（Android 12 以降）

### iOS のフィードバックタイプ

- Impact 系
  - `ImpactLight` / `ImpactMedium` / `ImpactHeavy`
  - `ImpactSoft` / `ImpactRigid`
- Notification 系
  - `Success` / `Warning` / `Error`
- `SelectionChanged`: ピッカーなどの選択変更時

## ベストプラクティス

### 1. システム標準パターンの適切な使用

:::message
OS のガイドラインに沿ったフィードバック種類を選び、一貫した体験を提供しましょう。
:::

### 2. 頻度と強さの調整

- 頻繁な操作 → 短く弱い振動
- 重要な操作 → やや強い振動
- 過剰な振動は避ける

### 3. 一貫性の維持

- 同じ操作には同じ Haptic を使用
- プラットフォームの慣習に合わせる

### 4. 視覚・聴覚効果との調和

- アニメーションやサウンドと同期
- 触覚単独での使用は避ける

### 5. ユーザー設定への配慮

:::message
システムの振動設定を OFF にしているユーザーへの配慮として、アプリ内で ON/OFF 切り替えを提供することを検討しましょう。
:::

## おわりに

Haptic 機能は画面上の操作を物理的な感覚へと拡張する強力な手段です。適切に導入することで、アプリの UX を大きく向上させることができます。

## 参考リンク

- [Cordova 公式ドキュメント](https://cordova.apache.org/)
- [cordova-plugin-haptic (GitHub)](https://github.com/example/cordova-plugin-haptic)
- [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Android 開発者ドキュメント（Haptic 関連）](https://developer.android.com/guide/topics/ui/haptic-feedback)
