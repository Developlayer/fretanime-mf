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
- focusDistance範囲: 0.0〜2.0（メートル単位）

### 分割モード
1. **田んぼ（grid）**: 2x2の4分割
2. **タテ短冊（column）**: 縦4分割
3. **ヨコ短冊（row）**: 横4分割

### アニメーション仕様
- フレーム間隔: 200ms
- 順序: 0→1→2→3→0→...（無限ループ）

### 動画出力
- 形式: WebM
- フレームレート: 30fps
- ファイル名: fretanime_[timestamp].webm

## ファイル構成
```
fretanime-mf/
├── index.html          # メインアプリ（単一ファイル）
├── CLAUDE.md           # このファイル（開発メモ）
├── Development-log.md  # 開発ログ
└── fretanime-mf_specification_v1.1.md  # 仕様書
```

## 開発時の注意点
- コメントは日本語で記述
- エラーハンドリング必須
- UIは右上にオーバーレイ表示
- 画面全体をCanvasで覆う（フルスクリーン前提）
