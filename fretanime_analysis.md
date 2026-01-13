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
| 再生ボタン | アニメーション開始（枠内を順番に表示） |
| 停止ボタン | アニメーション停止 |
| 録画ボタン | 動画ファイル書き出し |

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
// フレームバッファ（各区画の映像を保存）
const frameBuffers = [null, null, null, null];
let captureIndex = 0;

// 区画のキャプチャ
function captureFrame(gridMode, index) {
  const tempCanvas = document.createElement('canvas');
  const tempCtx = tempCanvas.getContext('2d');
  
  let sx, sy, sw, sh; // ソース座標
  
  switch(gridMode) {
    case 'grid': // 2x2
      sw = video.videoWidth / 2;
      sh = video.videoHeight / 2;
      sx = (index % 2) * sw;
      sy = Math.floor(index / 2) * sh;
      break;
    case 'column': // 縦4分割
      sw = video.videoWidth / 4;
      sh = video.videoHeight;
      sx = index * sw;
      sy = 0;
      break;
    case 'row': // 横4分割
      sw = video.videoWidth;
      sh = video.videoHeight / 4;
      sx = 0;
      sy = index * sh;
      break;
  }
  
  tempCanvas.width = sw;
  tempCanvas.height = sh;
  tempCtx.drawImage(video, sx, sy, sw, sh, 0, 0, sw, sh);
  
  return tempCanvas;
}
```

### 4.4 アニメーション再生

```javascript
let isPlaying = false;
let currentFrame = 0;
let lastFrameTime = 0;
const FRAME_DURATION = 200; // ミリ秒（調整可能）

function animate(timestamp) {
  if (!isPlaying) return;
  
  if (timestamp - lastFrameTime > FRAME_DURATION) {
    // 次のフレームを表示
    displayFrame(frameBuffers[currentFrame]);
    currentFrame = (currentFrame + 1) % 4;
    lastFrameTime = timestamp;
  }
  
  requestAnimationFrame(animate);
}

function displayFrame(frameCanvas) {
  // 全画面表示
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(frameCanvas, 0, 0, canvas.width, canvas.height);
}
```

### 4.5 動画録画

```javascript
let mediaRecorder;
let recordedChunks = [];

function startRecording() {
  const stream = canvas.captureStream(30); // 30 FPS
  
  const options = {
    mimeType: 'video/webm;codecs=vp9'
  };
  
  // ブラウザサポート確認
  if (!MediaRecorder.isTypeSupported(options.mimeType)) {
    options.mimeType = 'video/webm';
  }
  
  mediaRecorder = new MediaRecorder(stream, options);
  
  mediaRecorder.ondataavailable = (e) => {
    if (e.data.size > 0) {
      recordedChunks.push(e.data);
    }
  };
  
  mediaRecorder.onstop = () => {
    downloadVideo();
  };
  
  mediaRecorder.start();
}

function stopRecording() {
  mediaRecorder.stop();
}

function downloadVideo() {
  const blob = new Blob(recordedChunks, { type: 'video/webm' });
  const url = URL.createObjectURL(blob);
  
  const a = document.createElement('a');
  a.href = url;
  a.download = 'fretanime_' + Date.now() + '.webm';
  a.click();
  
  URL.revokeObjectURL(url);
  recordedChunks = [];
}
```

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
  currentFrameIndex: 0,
  frameBuffers: [null, null, null, null],
  
  // 録画状態
  isRecording: false,
  recordedChunks: []
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
- [ ] フレームバッファリング
- [ ] アニメーション再生/停止
- [ ] 動画録画・ダウンロード

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

1. **具体的なフレームレート設定**
2. **UIライブラリの詳細**
3. **状態管理ライブラリ（Redux等）の使用有無**
4. **PWA対応の有無**
5. **エラーハンドリングの詳細**
6. **アナリティクス実装**

これらは実際にブラウザのDevToolsで調査するか、開発者に問い合わせる必要があります。

---

*このドキュメントは https://www.fretanime.jp/ および https://www.fretanime.jp/app/ の公開情報に基づいて作成されました。*
