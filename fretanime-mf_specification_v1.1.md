# fretanime-mf 開発仕様書

**Claude Code向け 完全実装ガイド**

作成日: 2025年1月14日
バージョン: 1.2（2026年1月14日更新）

---

## 1. プロジェクト概要

### 1.1 目的
本プロジェクトは、既存の「フレットアニメ」Webアプリをクローンし、マニュアルフォーカス機能を追加した改良版を開発することを目的とする。

### 1.2 アプリケーション名
**fretanime-mf**（mf = Manual Focus）

### 1.3 オリジナルアプリ
参照元: https://www.fretanime.jp/app/

### 1.4 基本機能（オリジナルと同等）
- カメラ映像をリアルタイムで分割表示
- 3種類の分割モード（田んぼ2x2、タテ短冊、ヨコ短冊）
- 4フレームを順番に切り替えてアニメーション再生
- 動画書き出し機能（MP4形式、1ループ自動撮影）

### 1.5 新規追加機能
- AF（オートフォーカス）/ MF（マニュアルフォーカス）切り替え
- MF時のフォーカス距離調整（2cm〜100cm）

---

## 2. ターゲット環境

### 2.1 対象デバイス

| 項目 | 詳細 |
|------|------|
| 機種 | NEC Chromebook Y2 |
| OS | ChromeOS |
| ブラウザ | Google Chrome（最新版）**フルスクリーンモード（F11）** |
| 画面解像度 | 1366 x 768 ピクセル |
| アスペクト比 | 16:9 |

### 2.2 表示モード

> ⚠️ **重要**: 本アプリは**フルスクリーン表示を前提**として設計する。

- ブラウザのF11キーでフルスクリーンにして使用
- カメラ映像が画面全体（1366x768）を覆う
- アドレスバーやタブは表示されない状態を想定
- フルスクリーン時に綺麗に4分割（または4短冊）になるようレイアウト

### 2.3 異なる端末での画面対応

本アプリは**Chromebook固定ではなく、動的に画面サイズに対応**する設計になっている。

**実装方式:**
- Canvasサイズは `window.innerWidth` / `window.innerHeight` で取得
- ウィンドウリサイズ時に自動で再計算
- グリッドラインは画面サイズに対する比率（50%、25%等）で描画

**各端末での動作:**

| 端末 | 解像度 | アスペクト比 | 動作 |
|------|--------|-------------|------|
| Chromebook Y2 | 1366×768 | 16:9 | ほぼ全画面ピッタリ |
| フルHDモニター | 1920×1080 | 16:9 | 全画面表示（拡大） |
| 4Kモニター | 3840×2160 | 16:9 | 全画面表示（大幅拡大、画質低下の可能性） |
| iPad (横) | 1024×768 | 4:3 | 上下がカット（横に合わせる） |
| iPhone (縦) | 390×844 | 縦長 | 左右が大幅カット |
| ウルトラワイド | 2560×1080 | 21:9 | 左右がカット |

**注意点:**
- カメラ映像は16:9（1280×720）を要求するため、画面のアスペクト比が異なる場合は**カバー表示**（余白なし、一部カット）となる
- 出力動画（MP4）の解像度は**表示時のCanvasサイズに依存**する
  - 大きな画面で録画 → 高解像度の動画
  - 小さな画面で録画 → 低解像度の動画
- 4分割の位置は画面サイズに対する比率で計算されるため、どの端末でも正しく分割される

### 2.4 カメラ要件

**【重要】** NEC Y2には2つのカメラが搭載されている：

| カメラ位置 | 用途 | 使用 |
|-----------|------|------|
| 画面側（インカメラ） | ビデオ通話用 | ❌ 使用しない |
| キーボード側（アウトカメラ） | 物撮り・書画カメラ用 | ✅ **こちらを使用** |

> ⚠️ **注意**: 必ずアウトカメラ（キーボード側、`facingMode: 'environment'`）を起動すること。デフォルトのインカメラが起動しないよう実装すること。

---

## 3. 技術仕様

### 3.1 使用技術スタック

| カテゴリ | 技術 | 備考 |
|---------|------|------|
| 言語 | HTML5, CSS3, JavaScript (ES6+) | フレームワーク不要 |
| カメラAPI | MediaDevices.getUserMedia() | WebRTC標準API |
| 描画 | Canvas API | 2D Context使用 |
| 録画 | WebCodecs API + mp4-muxer | MP4形式出力 |
| アニメーション | requestAnimationFrame | 60fps対応 |

### 3.2 フォーカス制御API

`MediaStreamTrack.applyConstraints()`を使用してフォーカスを制御する。

**オートフォーカス（AF）設定:**
```javascript
track.applyConstraints({ advanced: [{ focusMode: 'continuous' }] })
```

**マニュアルフォーカス（MF）設定:**
```javascript
track.applyConstraints({ advanced: [{ focusMode: 'manual', focusDistance: value }] })
```

**focusDistance値の範囲:**

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| 最小値 | 0.02（2cm） | 最も近い位置（マクロ） |
| 最大値 | 1.0（100cm） | 最も遠い位置 |
| 単位 | メートル | 0.01 = 1cm |

> 💡 **実装のヒント**: カメラの能力を事前に`getCapabilities()`で取得し、`focusDistance`のmin/maxを確認すること。サポートされていない場合はUIを無効化する。

### 3.3 動画書き出し仕様

| 項目 | 仕様 |
|------|------|
| 出力形式 | MP4 (video/mp4) |
| コーデック | H.264 (AVC) |
| フレームレート | 4fps（1秒4フレーム） |
| ファイル名 | fretanime_[YYYYMMDD_HHMMSS].mp4 |
| 録画方式 | 1ループ自動撮影（ボタン押下で4フレーム撮影） |

> **実装メモ**: WebCodecs API (VideoEncoder) + mp4-muxer ライブラリを使用してMP4を生成。

---

## 4. UI/UX仕様

### 4.1 全体レイアウト（最重要）

> ⚠️ **重要**: オリジナルのフレットアニメと同じレイアウトを厳密に再現すること。

**レイアウトの特徴:**
- **カメラ映像が画面全体を覆う**（フルスクリーン前提）
- **UIは描画領域の上にオーバーレイ表示**（別領域ではない）
- **UIは右上に配置**
- UIの背景は半透明で、カメラ映像が透けて見える

**画面構成イメージ:**
```
┌─────────────────────────────────────────────────────────┐
│                                    ┌──────────────────┐ │
│                                    │ [フォーカスUI]   │ │
│                                    │ [田][タテ][ヨコ] │ │
│      カメラ映像（フルスクリーン）      │ [▶][⏹][⏺]     │ │
│      Canvas が画面全体を覆う         └──────────────────┘ │
│                                                         │
│         ┌─────────┬─────────┐                           │
│         │    0    │    1    │  ← グリッドライン         │
│         ├─────────┼─────────┤    （プレビュー時表示）    │
│         │    2    │    3    │                           │
│         └─────────┴─────────┘                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.2 UIの配置とオーバーレイ

**UIコンテナの特徴:**
- **position: absolute** または **position: fixed** で右上に固定
- **背景: 半透明**（例: `rgba(0, 0, 0, 0.5)`）
- **角丸**で柔らかい印象
- **z-index** でカメラ映像の上に表示

**配置順序（左から右）:**
```
[AF/MF切替] [フォーカススライダー] [距離表示] | [田] [タテ] [ヨコ] | [▶/⏹] [⏺]
└──────── フォーカスUI（新規追加）────────┘   └── グリッド切替 ──┘   └─ 再生/録画 ─┘
```

> **注**: 再生/停止ボタンは1つに統合（トグル式）

### 4.3 フォーカスコントロールUI（新規追加）

**配置位置:** 既存ボタン群の**左側**（右上のオーバーレイUI内）

**UI要素:**

| 要素 | 種類 | 動作 |
|------|------|------|
| AF/MFトグルボタン | ボタン | クリックでAF⇄MF切替。現在のモードを表示 |
| フォーカススライダー | range input | MF時のみ有効。2cm〜100cmを調整（幅400px） |
| 距離表示 | テキスト | 現在のフォーカス距離を数値で表示（例：50cm） |

**状態による表示変化:**

| 状態 | トグルボタン表示 | スライダー | 距離表示 |
|------|-----------------|-----------|----------|
| AFモード | 「AF」（アクティブ色） | 無効（グレーアウト） | 「AUTO」 |
| MFモード | 「MF」（アクティブ色） | 有効（操作可能） | 「XXcm」 |

### 4.4 既存ボタン仕様（オリジナル踏襲）

| ボタン | アイコン/テキスト | 機能 |
|--------|------------------|------|
| グリッドボタン | 田/タテ/ヨコ | 分割モード切替 |
| 再生/停止ボタン | ▶/⏹ | アニメーション開始/停止（トグル式） |
| 録画ボタン | ⏺ | 1ループ（4フレーム）自動撮影・保存 |

### 4.5 デザイン仕様

オリジナルのフレットアニメのデザインを踏襲する。

**全体:**
- カメラ映像が画面全体を覆う
- UIは右上にオーバーレイ

**UIコンテナ:**
- 背景: 半透明ダーク（`rgba(0, 0, 0, 0.6)` 程度）
- 角丸: `border-radius: 8px` 程度
- パディング: 適度な余白

**ボタン:**
- 丸みのある角
- 適度なパディング
- アクティブ状態: 明るい色でハイライト

**フォント:**
- システムフォント（-apple-system, sans-serif）
- 白文字（カメラ映像上で視認しやすく）

---

## 5. 機能仕様詳細

### 5.1 カメラ初期化シーケンス

1. `getUserMedia()`でカメラアクセスをリクエスト
2. `facingMode: 'environment'`でアウトカメラを指定
3. 解像度は1280x720を理想値として指定
4. ストリーム取得後、Videoタグに接続
5. `getCapabilities()`でフォーカス制御のサポートを確認
6. サポートされていればフォーカスUIを有効化、なければ無効化

### 5.2 フルスクリーン対応

**Canvas サイズ設定:**
```javascript
// 画面全体を覆うように設定
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

// ウィンドウリサイズ時に追従
window.addEventListener('resize', () => {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
});
```

**CSSでの全画面表示:**
```css
body {
  margin: 0;
  padding: 0;
  overflow: hidden;  /* スクロール禁止 */
}

#canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
}
```

### 5.3 グリッド分割ロジック

**田んぼモード（Grid）**
```
[0] [1]    インデックス配置
[2] [3]

フレーム0: (0, 0) から (width/2, height/2)
フレーム1: (width/2, 0) から (width, height/2)
フレーム2: (0, height/2) から (width/2, height)
フレーム3: (width/2, height/2) から (width, height)
```

**タテ短冊モード（Column）**
```
[0] [1] [2] [3]    左から右へ

フレームN: (width/4 * N, 0) から (width/4 * (N+1), height)
```

**ヨコ短冊モード（Row）**
```
[0]
[1]    上から下へ
[2]
[3]

フレームN: (0, height/4 * N) から (width, height/4 * (N+1))
```

### 5.4 グリッドライン表示

プレビューモード（アニメーション停止中）では、分割位置を示すグリッドラインを表示する。

```javascript
function drawGridLines(ctx, mode) {
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
  ctx.lineWidth = 2;
  
  const w = canvas.width;
  const h = canvas.height;
  
  if (mode === 'grid') {
    // 縦線
    ctx.beginPath();
    ctx.moveTo(w/2, 0);
    ctx.lineTo(w/2, h);
    ctx.stroke();
    // 横線
    ctx.beginPath();
    ctx.moveTo(0, h/2);
    ctx.lineTo(w, h/2);
    ctx.stroke();
  } else if (mode === 'column') {
    // 縦線3本
    for (let i = 1; i < 4; i++) {
      ctx.beginPath();
      ctx.moveTo(w/4 * i, 0);
      ctx.lineTo(w/4 * i, h);
      ctx.stroke();
    }
  } else if (mode === 'row') {
    // 横線3本
    for (let i = 1; i < 4; i++) {
      ctx.beginPath();
      ctx.moveTo(0, h/4 * i);
      ctx.lineTo(w, h/4 * i);
      ctx.stroke();
    }
  }
}
```

### 5.5 アニメーション再生

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| フレーム間隔 | 250ms | 1周1秒（4フレーム×250ms） |
| ループ | 無限ループ | 停止ボタンで終了 |
| 表示順序 | 0→1→2→3→0→... | インデックス順に繰り返し |
| 表示方法 | 等倍で中央表示 | 引き伸ばさず、周囲は黒 |

---

## 6. エラーハンドリング

### 6.1 想定されるエラーと対処

| エラー | 原因 | 対処方法 |
|--------|------|----------|
| カメラアクセス拒否 | ユーザーが許可しなかった | メッセージ表示「カメラへのアクセスを許可してください」 |
| カメラが見つからない | デバイスにカメラがない | メッセージ表示「カメラが見つかりません」 |
| フォーカス制御非対応 | カメラがMFをサポートしない | フォーカスUIを無効化、AFのみで動作 |
| 録画非対応 | ブラウザがMediaRecorderをサポートしない | 録画ボタンを無効化 |

### 6.2 フォールバック動作

フォーカス制御がサポートされていない場合：
1. AF/MFトグルボタンをグレーアウト
2. スライダーを非表示または無効化
3. その他の機能は正常に動作させる

---

## 7. ファイル構成

### 7.1 推奨ディレクトリ構造

```
fretanime-mf/
├── index.html          # メインHTML（全てを含む単一ファイル推奨）
├── README.md           # プロジェクト説明
└── vercel.json         # Vercel設定（任意）
```

> 💡 **推奨**: 単一のindex.htmlファイルに全てを含める構成。シンプルで管理しやすい。

### 7.2 index.htmlの基本構造

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>fretanime-mf</title>
    <style>
        /* ===== リセット & 基本スタイル ===== */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            overflow: hidden;  /* スクロール禁止 */
            background: #000;
        }
        
        /* ===== カメラ映像（フルスクリーン） ===== */
        #video {
            display: none;  /* 非表示（Canvasに描画するため） */
        }
        
        #canvas {
            position: fixed;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
        }
        
        /* ===== UIオーバーレイ（右上） ===== */
        .ui-container {
            position: fixed;
            top: 20px;
            right: 20px;
            display: flex;
            gap: 10px;
            background: rgba(0, 0, 0, 0.6);
            padding: 10px 15px;
            border-radius: 8px;
            z-index: 100;
        }
        
        /* ===== ボタン共通 ===== */
        button {
            padding: 10px 15px;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 14px;
            font-weight: bold;
            color: white;
            background: rgba(255, 255, 255, 0.2);
            transition: background 0.2s;
        }
        
        button:hover {
            background: rgba(255, 255, 255, 0.3);
        }
        
        button.active {
            background: #4a90d9;
        }
        
        button:disabled {
            opacity: 0.4;
            cursor: not-allowed;
        }
        
        /* ===== フォーカスコントロール ===== */
        .focus-control {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        
        .focus-slider {
            width: 400px;
        }
        
        .focus-value {
            min-width: 50px;
            color: white;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <!-- カメラ映像（非表示） -->
    <video id="video" autoplay playsinline muted></video>
    
    <!-- 描画Canvas（フルスクリーン） -->
    <canvas id="canvas"></canvas>
    
    <!-- UIオーバーレイ（右上） -->
    <div class="ui-container">
        <!-- フォーカスコントロール（新規追加） -->
        <div class="focus-control">
            <button id="focusModeBtn">AF</button>
            <input type="range" id="focusSlider" class="focus-slider"
                   min="2" max="100" value="50" disabled>
            <span id="focusValue" class="focus-value">AUTO</span>
        </div>
        
        <!-- 区切り -->
        <div style="width: 1px; background: rgba(255,255,255,0.3);"></div>
        
        <!-- グリッドモード -->
        <button id="gridBtn" class="active" data-mode="grid">田</button>
        <button id="columnBtn" data-mode="column">タテ</button>
        <button id="rowBtn" data-mode="row">ヨコ</button>
        
        <!-- 区切り -->
        <div style="width: 1px; background: rgba(255,255,255,0.3);"></div>
        
        <!-- 再生/録画 -->
        <button id="playStopBtn">▶</button>
        <button id="recordBtn">⏺</button>
    </div>
    
    <script>
        /* JavaScript をここに記述 */
    </script>
</body>
</html>
```

---

## 8. デプロイ手順

### 8.1 使用サービス

| 項目 | 内容 |
|------|------|
| サービス名 | Vercel (https://vercel.com) |
| プラン | Hobby（無料） |
| 帯域幅 | 月間100GB（30人同時利用に十分） |
| 自動デプロイ | GitHubリポジトリ連携 |

### 8.2 デプロイ手順

1. GitHubにリポジトリを作成（例：fretanime-mf）
2. コードをリポジトリにプッシュ
3. Vercelにサインアップ（GitHubアカウントで連携）
4. 「New Project」からGitHubリポジトリを選択
5. デフォルト設定のまま「Deploy」をクリック
6. 自動生成されたURL（例：fretanime-mf.vercel.app）でアクセス可能

> ℹ️ **自動更新**: 以降、GitHubにプッシュするたびに自動でVercelにデプロイされる。

### 8.3 HTTPS要件

カメラAPIはHTTPS環境でのみ動作する。Vercelは自動的にHTTPSを提供するため、追加設定は不要。（localhostでの開発時は例外的にHTTPでも動作する）

---

## 9. テスト項目

### 9.1 必須テスト項目

| テスト項目 | 確認内容 | 期待結果 |
|-----------|---------|----------|
| カメラ起動 | アプリ起動時にカメラが起動するか | アウトカメラの映像が表示される |
| カメラ選択 | 正しいカメラが選択されているか | キーボード側カメラが使用される |
| フルスクリーン | F11でフルスクリーンにできるか | 画面全体にカメラ映像が表示 |
| グリッド表示 | 4分割が綺麗に表示されるか | 画面を均等に4分割 |
| グリッド切替 | 3つのモードが切り替わるか | 田/タテ/ヨコが正常に切り替わる |
| アニメーション | 再生/停止が機能するか | フレームが順番に切り替わる |
| AF/MF切替 | フォーカスモードが切り替わるか | UIの状態が変わり、実際に制御される |
| MFスライダー | フォーカス距離が変わるか | スライダー操作でピントが変化 |
| 動画録画 | 録画と保存ができるか | MP4ファイルがダウンロードされる |
| UIオーバーレイ | UIが映像の上に表示されるか | 右上に半透明UIが重なる |

---

## 10. Claude Code向け実装指示

> ⚠️ **重要**: このセクションはClaude Codeが実装する際の具体的な指示です。

### 10.1 実装の優先順位

1. **Step 1:** 基本的なHTML/CSS構造を作成（フルスクリーン対応）
2. **Step 2:** カメラ起動機能（アウトカメラ指定）
3. **Step 3:** Canvas描画（画面全体を覆う）とグリッドライン
4. **Step 4:** グリッド分割とアニメーション再生/停止
5. **Step 5:** 動画録画機能
6. **Step 6:** フォーカス制御機能（AF/MF）

### 10.2 重要な実装ポイント

#### カメラ設定（最重要）

```javascript
const constraints = {
  video: {
    facingMode: { exact: 'environment' },  // 必ずアウトカメラを指定
    width: { ideal: 1280 },
    height: { ideal: 720 }
  },
  audio: false
};

// facingMode: 'environment' だけだとフォールバックされる可能性があるため
// { exact: 'environment' } を使用すること
```

#### フルスクリーン対応（最重要）

```javascript
// Canvas を画面全体に設定
function resizeCanvas() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}

window.addEventListener('resize', resizeCanvas);
window.addEventListener('load', resizeCanvas);
```

```css
/* フルスクリーン用CSS */
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background: #000;
}

#canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
}
```

#### UIオーバーレイ

```css
/* 右上に固定配置 */
.ui-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 100;
  background: rgba(0, 0, 0, 0.6);
  /* ... */
}
```

#### フォーカス制御

```javascript
// カメラの能力を確認
const track = stream.getVideoTracks()[0];
const capabilities = track.getCapabilities();

if (capabilities.focusMode) {
  // フォーカス制御がサポートされている
  const supportedModes = capabilities.focusMode;
  const hasMF = supportedModes.includes('manual');
  
  if (hasMF && capabilities.focusDistance) {
    // MFがサポートされている場合、focusDistanceの範囲を取得
    const minDist = capabilities.focusDistance.min;
    const maxDist = capabilities.focusDistance.max;
    // スライダーの範囲を設定
  }
}

// フォーカス設定の適用
async function setFocusMode(mode, distance = null) {
  const constraints = { advanced: [{ focusMode: mode }] };
  if (mode === 'manual' && distance !== null) {
    constraints.advanced[0].focusDistance = distance;
  }
  await track.applyConstraints(constraints);
}
```

### 10.3 コード品質要件

- コメントは日本語で記述すること
- エラーハンドリングを必ず実装すること
- console.logでデバッグ情報を出力すること（開発中）
- 変数名・関数名は英語で、意味が分かりやすいものにすること

---

## 付録: 参考リソース

### A.1 オリジナルアプリ
- フレットアニメ公式: https://www.fretanime.jp/
- Chromebook版アプリ: https://www.fretanime.jp/app/

### A.2 技術ドキュメント
- MediaDevices.getUserMedia() - MDN
- MediaStreamTrack.applyConstraints() - MDN
- Canvas API - MDN
- MediaRecorder API - MDN
- ImageCapture.getPhotoCapabilities() - W3C (フォーカス制御参考)

### A.3 デプロイサービス
- Vercel: https://vercel.com/
- Vercel ドキュメント: https://vercel.com/docs

---

*— 仕様書 終わり —*
