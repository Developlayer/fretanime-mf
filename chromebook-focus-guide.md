# Chromebook カメラ マニュアルフォーカス実装ガイド

作成日: 2026年2月17日

fretanime-mf プロジェクトで判明した、Chrome + Chromebook（NEC Y2）環境でのフォーカス制御に関する知見をまとめたドキュメントです。

---

## 1. 最重要: `advanced` オプションで focusMode と focusDistance を同時設定する

**focusModeとfocusDistanceを別々にapplyConstraintsすると、ChromebookではAFが維持されます。**

```javascript
// ❌ ダメな方式（別々に設定 → AFが維持される）
await track.applyConstraints({ focusMode: 'manual' });
await track.applyConstraints({ focusDistance: value });

// ❌ これもダメ（advancedなしのフラット指定）
await track.applyConstraints({ focusMode: 'manual', focusDistance: value });

// ✅ 正しい方式（advancedオプションで同時設定）
await track.applyConstraints({
    advanced: [{
        focusMode: 'manual',
        focusDistance: value
    }]
});
```

`advanced` 配列の中に1つのオブジェクトとして `focusMode` と `focusDistance` を**両方入れる**ことが必須です。

---

## 2. getUserMedia ではフォーカスモードを指定しない

カメラ取得時（`getUserMedia`）に `focusMode: 'manual'` を含めると、**Chromebookではカメラ取得自体が失敗します**。

```javascript
// ❌ ダメ（Chromebookでカメラ取得失敗）
const stream = await navigator.mediaDevices.getUserMedia({
    video: {
        facingMode: { exact: 'environment' },
        focusMode: 'manual'  // これをつけるとダメ
    }
});

// ✅ 正しい方式（まずカメラ取得、フォーカスは後から設定）
const stream = await navigator.mediaDevices.getUserMedia({
    video: {
        facingMode: { exact: 'environment' },
        width: { ideal: 1280 },
        height: { ideal: 720 }
    },
    audio: false
});

// 取得後にapplyConstraintsでフォーカス設定
const track = stream.getVideoTracks()[0];
await track.applyConstraints({
    advanced: [{ focusMode: 'manual', focusDistance: 5 }]
});
```

---

## 3. 2段階フォーカス設定（レンズを確実に動かすテクニック）

AFで合わせたピント位置と目標距離が近い場合、`focusMode: 'manual'` に設定しても**レンズが物理的に動かず、AFで合わせた位置がそのまま固定される**問題があります。

**解決策: 一度わざと逆方向に動かしてから目標に合わせる**

```javascript
// ∞（遠景固定）に設定したい場合の例

// ステップ1: 近距離に設定してレンズを手前に動かす
await track.applyConstraints({
    advanced: [{ focusMode: 'manual', focusDistance: 0.02 }]  // 最短距離
});
await new Promise(resolve => setTimeout(resolve, 300));  // レンズ移動を待つ

// ステップ2: ∞に設定してレンズを奥に動かす
await track.applyConstraints({
    advanced: [{ focusMode: 'manual', focusDistance: 5 }]  // 最遠距離
});
```

**なぜ必要か:** カメラ起動直後はAFが動作しており、例えば机の上の物にピントが合っている状態です。ここでいきなり `focusDistance: 5`（∞）を設定しても、AFがすでに近い位置にピントを合わせていた場合、レンズが動かないケースがありました。一度最短距離に振ることで、レンズを確実に物理的に動かせます。

---

## 4. AF自動復帰問題と watchdog

Chromebookでは、`focusMode: 'manual'` に設定しても**しばらくするとAF（continuous）に勝手に戻る**現象が発生します。

**解決策: 2秒間隔のwatchdog（監視タイマー）**

```javascript
function startFocusWatchdog() {
    setInterval(async () => {
        if (!track) return;

        const settings = track.getSettings();

        if (settings.focusMode !== 'manual') {
            // AFに戻っていたら再設定
            await track.applyConstraints({
                advanced: [{
                    focusMode: 'manual',
                    focusDistance: currentTargetDistance  // 現在の目標距離
                }]
            });
        }
    }, 2000);  // 2秒間隔
}
```

`getSettings().focusMode` を定期的にチェックし、`'manual'` でなくなっていたら即座に再設定します。

---

## 5. カメラ起動後に安定待機してからフォーカス設定する

カメラが完全に安定する前にフォーカスを設定しても効きません。video要素の `playing` イベント発火後、**さらに3秒待機**してからフォーカス設定を適用する必要があります。

```javascript
video.addEventListener('playing', () => {
    // 追加で3秒待機してカメラを完全に安定させる
    setTimeout(() => {
        applyInitialFocus();  // ここでフォーカス設定
    }, 3000);
});
```

---

## 6. リトライロジックが必要

初回の `applyConstraints` が成功しても、`getSettings()` で確認すると反映されていないことがあります。

```javascript
async function setFocusDistance(apiValue) {
    // 最大3回リトライ
    for (let attempt = 0; attempt < 3; attempt++) {
        try {
            await track.applyConstraints({
                advanced: [{
                    focusMode: 'manual',
                    focusDistance: apiValue
                }]
            });

            // 設定が反映されるまで待機
            await new Promise(resolve => setTimeout(resolve, 100));

            // 設定後の実際の値を確認
            const settings = track.getSettings();
            if (settings.focusMode === 'manual') {
                return true;  // 成功
            }

            // 失敗したら待機時間を増やして再試行
            await new Promise(resolve => setTimeout(resolve, 150));
        } catch (error) {
            await new Promise(resolve => setTimeout(resolve, 100));
        }
    }
    return false;  // 3回失敗
}
```

初回起動時のフォーカス適用は特に不安定なので、**最大10回試行**が有効です。

```javascript
async function applyInitialFocus() {
    const maxAttempts = 10;
    const intervalMs = 500;

    for (let attempt = 0; attempt < maxAttempts; attempt++) {
        // 2段階設定（セクション3参照）
        await track.applyConstraints({
            advanced: [{ focusMode: 'manual', focusDistance: 0.02 }]
        });
        await new Promise(resolve => setTimeout(resolve, 300));
        await track.applyConstraints({
            advanced: [{ focusMode: 'manual', focusDistance: 5 }]
        });

        await new Promise(resolve => setTimeout(resolve, intervalMs));

        if (track.getSettings().focusMode === 'manual') {
            break;  // 成功
        }
    }

    startFocusWatchdog();  // watchdog開始
}
```

---

## 7. getSettings() で検証するのは focusMode のみ

`focusDistance` の値は `getSettings()` で取得しても、設定した値と完全に一致しないことがあります。**focusDistanceの値で成否判定すると無限ループに陥ります。**

```javascript
// ✅ focusModeのみ検証
const settings = track.getSettings();
if (settings.focusMode === 'manual') {
    // 成功（focusDistanceの値は検証しない）
}

// ❌ focusDistanceも検証すると無限ループになる
if (settings.focusMode === 'manual' && settings.focusDistance === targetValue) {
    // この条件が永遠に満たされないことがある
}
```

---

## 8. getCapabilities() でカメラの能力を事前確認

フォーカス制御を実装する前に、カメラがマニュアルフォーカスをサポートしているか確認が必要です。

```javascript
const capabilities = track.getCapabilities();

// フォーカスモードのサポート確認
if (capabilities.focusMode) {
    const supportedModes = capabilities.focusMode;
    const hasMF = supportedModes.includes('manual');

    if (hasMF && capabilities.focusDistance) {
        // focusDistanceの範囲を取得
        const min = capabilities.focusDistance.min;   // 例: 0.02
        const max = capabilities.focusDistance.max;   // 例: 5
        const step = capabilities.focusDistance.step;  // 例: 0.01
        // → スライダーUIの範囲設定に使用
    }
}
```

NEC Chromebook Y2 での実測値:

| パラメータ | 値 |
|-----------|-----|
| focusDistance.min | 0.02（メートル） |
| focusDistance.max | 5（メートル） |
| focusDistance.step | 0.01 |
| focusMode | `['continuous', 'manual']` |

---

## 9. video要素を display:none にしない

Chromebookでは `display: none` のvideo要素は映像処理が正しく行われません。Canvasへの描画元として使うvideo要素は**画面外に配置**して非表示にします。

```css
/* ❌ ダメ（Chromebookでカメラ映像がCanvasに描画されない） */
#video {
    display: none;
}

/* ✅ 正しい方式（画面外に配置） */
#video {
    position: fixed;
    top: -9999px;
    left: -9999px;
    width: 1px;
    height: 1px;
    opacity: 0;
    pointer-events: none;
}
```

---

## 10. video.play() の失敗とユーザー操作リトライ

ChromeOSのautoplay制限により `video.play()` が失敗することがあります。失敗時はユーザーの操作（click / touchstart）後にリトライします。

```javascript
try {
    await video.play();
} catch (playError) {
    // ユーザーインタラクション後に再試行
    const retryPlay = async () => {
        try { await video.play(); } catch (e) {}
    };
    document.body.addEventListener('click', retryPlay, { once: true });
    document.body.addEventListener('touchstart', retryPlay, { once: true });
}
```

---

## 11. 3段階カメラフォールバック

`facingMode: { exact: 'environment' }` が失敗する環境もあるため、3段階のフォールバックが有効です。

```javascript
// 1段階目: exact environment（アウトカメラを厳密に指定）
try {
    return await navigator.mediaDevices.getUserMedia({
        video: { facingMode: { exact: 'environment' }, ... }
    });
} catch (e) {}

// 2段階目: prefer environment（優先指定）
try {
    return await navigator.mediaDevices.getUserMedia({
        video: { facingMode: 'environment', ... }
    });
} catch (e) {}

// 3段階目: 制約なし（どのカメラでも可）
return await navigator.mediaDevices.getUserMedia({
    video: { width: { ideal: 1280 }, height: { ideal: 720 } }
});
```

---

## 12. フォーカススライダーにはデバウンスをかける

スライダーを素早く動かすと `applyConstraints` が大量に発火して不安定になります。100msのデバウンスを入れます。

```javascript
let focusDebounceTimer = null;

slider.addEventListener('input', (event) => {
    const value = parseFloat(event.target.value);

    if (focusDebounceTimer) {
        clearTimeout(focusDebounceTimer);
    }

    focusDebounceTimer = setTimeout(async () => {
        await setFocusDistance(value);
    }, 100);
});
```

---

## 実装の全体フロー

```
1. getUserMedia() でカメラ取得（focusModeは指定しない）
2. video要素にストリーム接続（display:none は使わず画面外配置）
3. video.play() 実行（失敗時はユーザー操作リトライ）
4. playing イベント後、3秒待機（カメラ安定化）
5. getCapabilities() でMFサポートを確認
6. 初期フォーカス適用（2段階設定 × 最大10回リトライ）
7. watchdog 開始（2秒間隔で focusMode 監視・修復）
8. スライダー操作時は advanced で同時設定（デバウンス付き）
```

---

## ポイント早見表

| 項目 | やるべきこと |
|------|-------------|
| フォーカス設定 | `advanced` で focusMode と focusDistance を同時に |
| getUserMedia | focusModeは含めない（後から applyConstraints で設定） |
| レンズを確実に動かす | 一度逆方向（近距離）に振ってから目標距離に設定 |
| AF自動復帰対策 | 2秒間隔の watchdog で常時監視・修復 |
| 設定の成否判定 | focusMode のみ検証（focusDistance は検証しない） |
| カメラ安定化 | playing イベント後 3秒待機してからフォーカス設定 |
| リトライ | 通常操作: 最大3回、初回起動: 最大10回 |
| video要素 | display:none ではなく画面外配置 |
| 自動再生失敗 | click / touchstart でリトライ |
| スライダー | 100ms デバウンス |

---

*このドキュメントは fretanime-mf プロジェクト（NEC Chromebook Y2 + ChromeOS + Google Chrome）での実装経験に基づいています。*
