# CLAUDE.md - fretanime-mf 開発メモ

## プロジェクト概要
- **アプリ名**: fretanime-mf（mf = Manual Focus）
- **目的**: オリジナルの「フレットアニメ」をクローンし、マニュアルフォーカス機能を追加

## 重要な技術仕様

### ターゲット環境
- NEC Chromebook Y2
- ChromeOS / Google Chrome（フルスクリーンモード F11）
- 画面解像度: 1366 x 768px

### カメラ設定（最重要）
- **必ずアウトカメラを使用する**（キーボード側カメラ）
- `facingMode: { exact: 'environment' }` を使用
- インカメラ（画面側）は使用しない

### フォーカス制御
- AF: `focusMode: 'continuous'`
- MF: `focusMode: 'manual'` + `focusDistance: value`
- focusDistance範囲: 0.02〜1.0（メートル単位、2cm〜100cm）

### 分割モード
1. **田んぼ（grid）**: 2x2の4分割
2. **タテ短冊（column）**: 縦4分割
3. **ヨコ短冊（row）**: 横4分割

### アニメーション仕様
- フレーム間隔: 250ms（1周1秒）
- 順序: 0→1→2→3→0→...（無限ループ）
- 表示: 等倍で中央表示（引き伸ばさない）

### 動画出力
- 形式: MP4 (H.264)
- フレームレート: 4fps（1秒4フレーム）
- ファイル名: fretanime_[YYYYMMDD_HHMMSS].mp4
- 録画方式: 1ループ自動撮影（ボタン押下で4フレーム撮影）
- 実装: WebCodecs API + mp4-muxer

### mp4-muxer について
- バージョン: 5.1.3（CDN経由で読み込み）
- ライセンス: MIT
- セキュリティ: SRI (Subresource Integrity) 設定済み
- 状態: 非推奨（deprecated）だが、既知の脆弱性なし
- 後継: Mediabunny（必要になった場合に移行検討）

## コード構造

### 設定定数（CONFIG）
```javascript
CONFIG.DEBUG = false;  // true でデバッグログ有効化
CONFIG.FRAME_INTERVAL_MS = 250;  // アニメーション間隔
CONFIG.TOTAL_FRAMES = 4;  // フレーム数
```

### 状態管理（state）
```javascript
state.elements    // DOM要素のキャッシュ
state.camera      // カメラ関連の状態
state.animation   // アニメーション関連の状態
state.recording   // 録画関連の状態
state.display     // 表示設定
```

### デバッグモード
`CONFIG.DEBUG = true` に設定すると、コンソールに詳細ログが出力される。
本番環境では `false` に設定。

## ファイル構成
```
fretanime-mf/
├── index.html                         # メインアプリ（単一ファイル）
├── CLAUDE.md                          # このファイル（開発メモ）
├── Development-log.md                 # 開発ログ
├── fretanime-mf_specification_v1.1.md # 仕様書
└── fretanime_analysis.md              # 本家アプリ分析レポート
```

## 開発時の注意点
- コメントは日本語で記述
- エラーハンドリング必須
- UIは右上にオーバーレイ表示
- 画面全体をCanvasで覆う（フルスクリーン前提）

## 重要：変更してはいけない実装

以下はChromebook対応のために必須の実装。変更すると動作しなくなる可能性あり：

1. **video要素のCSS**: `display: none` ではなく画面外配置（`position: fixed; top: -9999px`）
2. **3段階カメラフォールバック**: exact → prefer → 制約なし
3. **mp4-muxer + WebCodecs API**: FFmpegは使用しない（SharedArrayBuffer問題）
4. **ユーザー操作リトライ**: video.play()失敗時のclick/touchstartリスナー
