# フレットアニメ 技術仕様分析レポート

## 1. アプリケーション概要

### 目的
「うごき」の面白さを体験する教育/クリエイティブツール。カメラ映像をリアルタイムで区切って表示し、アニメーション効果を観察するトレーニングを提供。

### 対象プラットフォーム
- **メイン**: Chromebook（`/app/`）
- **モバイル**: タブレット・スマートフォン（`/responsive/`）
- **ステータス**: β版

---

## 2. 機能仕様

### 2.1 コア機能

| 機能 | 説明 |
|------|------|
| **カメラキャプチャ** | デバイスカメラからリアルタイム映像取得 |
| **グリッド分割表示** | 映像を複数の区画に分割 |
| **アニメーション再生** | 区画を順番に切り替えてアニメーション効果を生成 |
| **動画書き出し** | アニメーションを動画ファイルとして保存 |

### 2.2 表示モード（3種類）

1. **田んぼモード（Grid）** - 2x2の4分割
2. **タテ短冊（Column）** - 縦に4分割
3. **ヨコ短冊（Row）** - 横に4分割

### 2.3 UIコントロール

| ボタン | 機能 |
|--------|------|
| グリッドボタン | 分割モード切替（田んぼ/タテ/ヨコ） |
| 再生/停止ボタン | アニメーション開始/停止（トグル式） |
| 録画ボタン | 1ループ（4フレーム）を自動撮影・保存 |

---

## 3. 推定技術スタック

### 3.1 フロントエンド

```
フレームワーク: Gatsby (SSG)
  └─ 根拠: /static/[hash]/[hash]/ 形式の画像パス構造
  
UIフレームワーク: React (Gatsbyの標準)
  
スタイリング: CSS (詳細不明)
```

### 3.2 使用API

| API | 用途 |
|-----|------|
| **MediaDevices API** | `navigator.mediaDevices.getUserMedia()` でカメラアクセス |
| **Canvas API** | 映像の描画・分割・合成処理 |
| **requestAnimationFrame** | アニメーションループ制御 |
| **MediaRecorder API** | 動画録画・書き出し |
| **Blob API** | 動画ファイル生成 |

---

## 4. 推定実装アーキテクチャ

### 4.1 映像取得フロー

```
[Camera] 
    ↓ getUserMedia()
[Video Element] (hidden)
    ↓ drawImage()
[Canvas Element]
    ↓ requestAnimationFrame loop
[Display Canvas]
```

### 4.2 コア処理の疑似コード

```javascript
// 1. カメラストリーム取得
const stream = await navigator.mediaDevices.getUserMedia({
  video: {
    facingMode: 'environment', // または 'user'
    width: { ideal: 1280 },
    height: { ideal: 720 }
  },
  audio: false
});

// 2. Video要素にストリームを設定
const video = document.getElementById('video');
video.srcObject = stream;
await video.play();

// 3. Canvasへの描画ループ
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

function drawFrame() {
  // カメラ映像をCanvasに描画
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
  requestAnimationFrame(drawFrame);
}
```

### 4.3 グリッド分割アルゴリズム

```javascript
// リアルタイム映像から直接分割表示（バッファリングなし）
// アニメーション中も常に最新のカメラ映像を使用

function drawFrame(gridMode, index, video, ctx, canvasWidth, canvasHeight) {
  let sx, sy, sw, sh; // ソース座標（ビデオ側）

  const vw = video.videoWidth;
  const vh = video.videoHeight;

  switch(gridMode) {
    case 'grid': // 2x2
      sw = vw / 2;
      sh = vh / 2;
      sx = (index % 2) * sw;
      sy = Math.floor(index / 2) * sh;
      break;
    case 'column': // 縦4分割
      sw = vw / 4;
      sh = vh;
      sx = index * sw;
      sy = 0;
      break;
    case 'row': // 横4分割
      sw = vw;
      sh = vh / 4;
      sx = 0;
      sy = index * sh;
      break;
  }

  // 等倍で中央に描画（引き伸ばさない）
  const drawX = (canvasWidth - sw) / 2;
  const drawY = (canvasHeight - sh) / 2;
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);
  ctx.drawImage(video, sx, sy, sw, sh, drawX, drawY, sw, sh);
}
```

> **注**: 本家はフレームバッファを使用せず、リアルタイム映像を直接分割表示する方式

### 4.4 アニメーション再生

```javascript
let isPlaying = false;
let currentFrame = 0;
let animationInterval = null;
const FRAME_DURATION = ???; // ミリ秒（本家の正確な値は未確認）

function startAnimation() {
  if (isPlaying) return;
  isPlaying = true;
  currentFrame = 0;

  // 一定間隔でフレームを切り替え
  animationInterval = setInterval(() => {
    currentFrame = (currentFrame + 1) % 4;
  }, FRAME_DURATION);
}

function stopAnimation() {
  isPlaying = false;
  if (animationInterval) {
    clearInterval(animationInterval);
    animationInterval = null;
  }
}

// 描画ループ内でcurrentFrameに基づいてリアルタイム映像を分割表示
// （フレームは等倍で中央表示、周囲は黒）
```

> **注**: 本家のFRAME_DURATIONの正確な値は確認できていない

### 4.5 動画録画

```javascript
// 1ループ自動撮影方式
// 録画ボタン押下で4フレーム（1ループ）を自動撮影・保存

async function recordOneLoop() {
  const frameInterval = FRAME_DURATION; // フレーム間隔
  const totalFrames = 4;
  const capturedFrames = [];

  // 4フレームを順番にキャプチャ
  for (let i = 0; i < totalFrames; i++) {
    currentFrame = i;
    await new Promise(resolve => setTimeout(resolve, 50)); // 描画待機

    // Canvasからフレームをキャプチャ
    const frameData = canvas.toDataURL('image/png');
    capturedFrames.push(frameData);

    if (i < totalFrames - 1) {
      await new Promise(resolve => setTimeout(resolve, frameInterval));
    }
  }

  // 動画として保存（形式は実装依存）
  await saveAsVideo(capturedFrames);
}

function downloadVideo(blob, filename) {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

> **注**: 録画は開始/停止のトグル方式ではなく、ボタン1回押下で1ループを自動撮影する方式

---

## 5. プロジェクト構造（推定）

```
fretanime/
├── src/
│   ├── components/
│   │   ├── App.jsx           # メインアプリケーション
│   │   ├── Camera.jsx        # カメラ制御
│   │   ├── Canvas.jsx        # 描画処理
│   │   ├── Controls.jsx      # UIコントロール
│   │   └── Recorder.jsx      # 録画機能
│   ├── hooks/
│   │   ├── useCamera.js      # カメラフック
│   │   ├── useAnimation.js   # アニメーションフック
│   │   └── useRecorder.js    # 録画フック
│   ├── utils/
│   │   └── gridUtils.js      # グリッド計算
│   └── pages/
│       ├── index.js          # ランディングページ
│       ├── app.js            # Chromebook版アプリ
│       └── responsive.js     # モバイル版アプリ
├── static/
│   ├── images/
│   │   ├── button_grid.png
│   │   ├── button_column.png
│   │   ├── button_row.png
│   │   ├── button_play.png
│   │   ├── button_stop.png
│   │   └── button_rec.png
│   └── movies/
│       └── sample.mp4
├── gatsby-config.js
└── package.json
```

---

## 6. 主要な状態管理

```javascript
// アプリケーション状態
const state = {
  // カメラ状態
  cameraReady: false,
  cameraStream: null,

  // 表示モード
  gridMode: 'grid',  // 'grid' | 'column' | 'row'

  // アニメーション状態
  isPlaying: false,
  currentFrameIndex: 0,  // 0-3
  // 注: フレームバッファは使用しない（リアルタイム表示）

  // 録画状態
  isRecording: false  // 1ループ撮影中フラグ
};
```

---

## 7. レスポンシブ対応（/responsive/）

モバイル版は以下の対応が必要と推測：

```javascript
// デバイス判定
const isMobile = /iPhone|iPad|Android/.test(navigator.userAgent);

// カメラ設定の調整
const constraints = {
  video: {
    // モバイルでは縦横比を考慮
    width: isMobile ? { ideal: 720 } : { ideal: 1280 },
    height: isMobile ? { ideal: 1280 } : { ideal: 720 },
    facingMode: isMobile ? 'environment' : 'user'
  }
};

// タッチイベント対応
canvas.addEventListener('touchstart', handleTouchStart);
canvas.addEventListener('touchmove', handleTouchMove);
```

---

## 8. 実装上の注意点

### 8.1 ブラウザ互換性

| 機能 | Chrome | Firefox | Safari | Edge |
|------|--------|---------|--------|------|
| getUserMedia | ✅ | ✅ | ✅ | ✅ |
| Canvas | ✅ | ✅ | ✅ | ✅ |
| MediaRecorder | ✅ | ✅ | ⚠️* | ✅ |
| captureStream | ✅ | ✅ | ⚠️* | ✅ |

*Safari 14.0+で部分的サポート

### 8.2 録画フォーマット

```javascript
// ブラウザ別の推奨MIMEタイプ
function getRecordingMimeType() {
  const types = [
    'video/webm;codecs=vp9',
    'video/webm;codecs=vp8',
    'video/webm',
    'video/mp4'
  ];
  
  for (const type of types) {
    if (MediaRecorder.isTypeSupported(type)) {
      return type;
    }
  }
  return '';
}
```

### 8.3 パフォーマンス最適化

- `requestAnimationFrame` による効率的なレンダリング
- オフスクリーンCanvasの活用
- 不要なリドローの回避
- メモリリーク防止（ストリームの適切な解放）

---

## 9. クローン開発のためのチェックリスト

### 必須機能
- [ ] カメラアクセス許可の取得
- [ ] リアルタイム映像表示
- [ ] 3種類のグリッド分割
- [ ] アニメーション再生/停止（リアルタイム分割表示）
- [ ] 動画録画・ダウンロード（1ループ自動撮影）

### 追加検討機能
- [ ] フレームレート調整
- [ ] 分割数のカスタマイズ
- [ ] フィルター/エフェクト
- [ ] 音声同時録音
- [ ] SNS共有機能
- [ ] ローカルストレージ保存

---

## 10. 参考リソース

### 公式ドキュメント
- [MediaDevices.getUserMedia() - MDN](https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/getUserMedia)
- [Canvas API - MDN](https://developer.mozilla.org/ja/docs/Web/API/Canvas_API)
- [MediaRecorder API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder)
- [requestAnimationFrame - MDN](https://developer.mozilla.org/ja/docs/Web/API/Window/requestAnimationFrame)

### フレームワーク
- [Gatsby](https://www.gatsbyjs.com/)
- [React](https://react.dev/)

---

## 11. 補足：確認できなかった詳細

以下の情報は公開情報からは確認できませんでした：

1. **具体的なフレーム間隔（FRAME_DURATION）の値**
2. **UIライブラリの詳細**
3. **状態管理ライブラリ（Redux等）の使用有無**
4. **PWA対応の有無**
5. **エラーハンドリングの詳細**
6. **アナリティクス実装**
7. **動画出力形式の詳細**

これらは実際にブラウザのDevToolsで調査するか、開発者に問い合わせる必要があります。

---

## 12. 2026年1月14日更新：動作確認結果

本家アプリの実際の動作を確認した結果、以下が判明：

| 項目 | 当初の推定 | 実際の動作 |
|------|-----------|-----------|
| フレーム表示方式 | バッファリング後に再生 | リアルタイム映像を直接分割表示 |
| アニメーション表示 | 全画面に引き伸ばし | 等倍で中央表示、周囲は黒 |
| 録画方式 | 開始/停止トグル | 1ループ自動撮影 |
| 再生/停止ボタン | 別々のボタン | 1つのトグルボタン |

---

*このドキュメントは https://www.fretanime.jp/ および https://www.fretanime.jp/app/ の公開情報と実際の動作確認に基づいて作成されました。*
