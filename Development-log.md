# Development Log - fretanime-mf

## 開発ログ

### 2026-01-13

#### 開発開始
- 仕様書（fretanime-mf_specification_v1.1.md）を確認
- プロジェクト構成を理解
- CLAUDE.mdと本ファイルを作成

#### index.htmlを作成（全機能実装完了）

**実装した機能:**

1. **HTML/CSS構造**
   - フルスクリーン対応レイアウト
   - 右上に半透明UIオーバーレイ
   - レスポンシブ対応（ウィンドウリサイズに追従）

2. **カメラ起動機能**
   - アウトカメラ（`facingMode: environment`）を優先使用
   - exact指定が失敗した場合のフォールバック処理
   - 1280x720の解像度で取得

3. **Canvas描画**
   - カメラ映像を画面全体に描画
   - アスペクト比を維持したカバー表示
   - 60fpsでの描画ループ

4. **グリッドライン表示**
   - 田んぼモード（2x2）: 縦線1本、横線1本
   - タテ短冊モード: 縦線3本
   - ヨコ短冊モード: 横線3本

5. **アニメーション再生/停止**
   - 200ms間隔でフレーム切り替え
   - 0→1→2→3→0...の無限ループ
   - 各フレームを画面全体に拡大表示

6. **動画録画機能**
   - WebM形式（VP9コーデック）で録画
   - 30fpsでCanvas映像をキャプチャ
   - 自動ダウンロード機能

7. **フォーカス制御機能**
   - AF/MF切り替えボタン
   - フォーカス距離スライダー（0cm〜200cm）
   - カメラの能力に応じてUIを有効/無効化

**ファイル構成:**
```
fretanime-mf/
├── index.html                        # メインアプリ（全機能含む）
├── CLAUDE.md                         # 開発メモ
├── Development-log.md                # 開発ログ（このファイル）
└── fretanime-mf_specification_v1.1.md # 仕様書
```

#### GitHubリポジトリ作成・Vercelデプロイ

※以下の作業はClaude Codeクラッシュ前に実施（ログ未記録だったため追記）

- GitHubにリポジトリを作成: `Developlayer/fretanime-mf`
- 初期コミット（v1.0）をプッシュ
- Vercelと連携してデプロイ
- デプロイURL: https://fretanime-mf.vercel.app/

---

### 2026-01-14

#### Chromebookでの動作テスト・バグ発見

NEC Chromebook Y2で動作確認を実施したところ、以下の問題が発生：

**症状:**
- カメラ権限を許可すると、カメラ横のランプは点灯（カメラアクセス成功）
- しかし画面にはグリッドラインとUIのみ表示され、カメラ映像が映らない（背景が真っ暗）

#### バグ修正

**原因分析:**

1. **video要素の`display: none`問題**
   - ChromeOSでは`display: none`のvideo要素は映像処理が正しく行われない可能性がある
   - ストリームは取得できているが、video要素が描画可能な状態にならない

2. **描画ループの開始タイミング問題**
   - 複数のイベント（onloadeddata, oncanplay, onplaying）で描画ループを開始しようとしていたが、条件分岐が複雑で開始されないケースがあった

3. **autoplay制限**
   - ChromeOSでのautoplay制限により`video.play()`が失敗する可能性

**修正内容:**

| 修正箇所 | 変更前 | 変更後 |
|----------|--------|--------|
| video要素のCSS | `display: none` | 画面外配置（position: fixed, top: -9999px） |
| video要素の属性 | なし | `width="1280" height="720"` 追加 |
| 描画ループ開始 | イベント待機後に開始 | 即座に開始、準備中メッセージ表示 |
| play()呼び出し | 直接呼び出し | loadedmetadata後に呼び出し |
| フォールバック | 2段階 | 3段階（exact → prefer → 制約なし） |
| ユーザー操作対応 | clickのみ | click + touchstart |

**コミット:** `31ddfdf` - fix: Chromebookでカメラ映像が表示されない問題を修正

**デプロイ:** GitHubプッシュ後、Vercelが自動デプロイ

#### UI改善・機能追加

Chromebookでのテスト後、以下の改善を実施：

**1. アニメーション間隔変更**
- 200ms → 250ms（1周1秒に変更）

**2. UIサイズ拡大（2倍）**
- ボタンパディング: 10px 15px → 20px 30px
- フォントサイズ: 14px → 28px
- スライダー幅: 100px → 200px
- スライダーつまみ: 16px → 32px
- 区切り線高さ: 24px → 48px

**3. フォーカス範囲変更**
- 0〜200cm → 2〜100cm

**4. 再生/停止ボタン統合**
- 2つのボタンを1つに統合
- 再生中は⏹（停止）、停止中は▶（再生）を表示

**5. アニメーション表示変更**
- 変更前: 分割フレームを画面全体に引き伸ばし
- 変更後: 分割フレームを等倍のまま画面中央に表示、周囲は黒

**6. 録画形式変更（WebM → MP4）**
- FFmpeg.wasmを使用してクライアントサイドでMP4変換
- 変換中は録画ボタンに「変換中...」を表示
- FFmpegが使用できない場合はWebMで保存

**コミット:** `34f5204` - feat: UI改善と機能追加

#### 追加修正（Chromebookテスト後）

**1. 録画ボタン機能変更**
- 変更前: 録画開始/停止のトグル方式
- 変更後: ボタン押下で1ループ（4フレーム×250ms = 1秒）を自動撮影・保存
- 撮影中はボタンに「撮影中」と表示

**2. MFスライダー拡大**
- 幅: 200px → 400px（2倍に拡大）
- より触りやすく操作しやすく

**3. MP4変換問題の修正**
- 問題: ChromebookでWebMのまま保存されてしまう
- 原因: FFmpeg.wasmがSharedArrayBufferを必要とするが、Chromebookでは無効
- 解決: シングルスレッド版FFmpeg（@ffmpeg/core-st）に変更

**コミット:** `869d6ca` - feat: 録画機能改善とUI調整

#### MP4録画の根本的修正

**問題:** FFmpeg.wasmがChromebookで動作せず、WebMのまま保存されてしまう

**解決策:** FFmpegを完全に廃止し、mp4-muxer + WebCodecs APIに変更

**変更内容:**
1. ライブラリ変更: FFmpeg.wasm → mp4-muxer
2. 録画方式変更: MediaRecorderストリーミング → フレームキャプチャ方式
3. エンコード: WebCodecs API（VideoEncoder）でH.264エンコード
4. タイムスタンプ形式: `1736850645123` → `20260114_153045`（年月日_時分秒）

**フレームキャプチャ方式の流れ:**
1. 各フレーム（0〜3）を順番に表示
2. Canvas.toDataURL()でPNG形式でキャプチャ
3. 4フレーム取得後、VideoEncoderでH.264エンコード
4. mp4-muxerでMP4コンテナに格納
5. Blobとしてダウンロード

**フォールバック:**
- WebCodecs APIがサポートされていない環境では静止画（PNG）として保存

**コミット:** `1ffcf8b` - feat: MP4録画を確実に動作するように改善

#### 本家アプリとの比較・ドキュメント修正

本家（https://www.fretanime.jp/app/）との違いを1つずつ確認し、以下が判明：

| 項目 | 当初の推定 | 実際の動作 | 修正要否 |
|------|-----------|-----------|---------|
| フレーム表示方式 | バッファリング後に再生 | リアルタイム映像を直接分割表示 | 現在の実装で正しい |
| アニメーション表示 | 全画面に引き伸ばし | 等倍で中央表示、周囲は黒 | 現在の実装で正しい |
| 録画方式 | 開始/停止トグル | 1ループ自動撮影 | 現在の実装で正しい |
| UIアイコン | 画像ファイル | - | 修正不要（Unicode文字で可） |
| フレームワーク | Gatsby + React | - | 修正不要（単一HTMLで可） |
| フォーカス制御 | なし | - | 維持（本プロジェクト独自機能） |

**ドキュメント修正内容:**

1. **CLAUDE.md**
   - フォーカス距離範囲: 0.0〜2.0m → 0.02〜1.0m（2cm〜100cm）

2. **fretanime-mf_specification_v1.1.md** → v1.2に更新
   - WebM → MP4（複数箇所）
   - フォーカス範囲: 0〜200cm → 2〜100cm
   - 再生/停止ボタン: 2つ → 1つ（トグル式）
   - 録画ボタン: 開始/停止 → 1ループ自動撮影
   - スライダー幅: 100px → 400px

3. **fretanime_analysis.md**
   - フレームバッファリング → リアルタイム表示に修正
   - 録画方式を1ループ自動撮影に修正
   - 再生/停止ボタンをトグル式に修正
   - 動作確認結果セクションを追加

#### コードリファクタリング（9項目）

コードの保守性・可読性・セキュリティを向上させるため、以下のリファクタリングを実施：

| # | 項目 | 内容 |
|---|------|------|
| 1 | 未使用変数の削除 | `mediaRecorder`, `recordedChunks`, `encodedChunks` を削除 |
| 2 | 定数の一元管理 | `CONFIG` オブジェクトに10個の設定を集約 |
| 3 | 関数名の修正 | `createGIFFromFrames` → `createFallbackImage`（実態に合わせて） |
| 4 | 状態管理のオブジェクト化 | `state` オブジェクトで14個の変数を整理 |
| 5 | initCamera()の分割 | 5つの小さな関数に分割（可読性向上） |
| 6 | デバッグログの制御 | `debugLog()` で `CONFIG.DEBUG` による制御 |
| 7 | DOM要素のキャッシュ | 6つのUI要素を起動時にキャッシュ（パフォーマンス向上） |
| 8 | ページ可視性のハンドリング | タブ切り替え時の省電力対応 |
| 9 | CDNスクリプトへのSRI追加 | mp4-muxerにintegrity属性を追加（セキュリティ強化） |

**リファクタリング後のコード構造:**

```javascript
// 設定定数
const CONFIG = {
    FRAME_INTERVAL_MS: 250,
    TOTAL_FRAMES: 4,
    VIDEO_WIDTH: 1280,
    VIDEO_HEIGHT: 720,
    FOCUS_MIN_M: 0.02,
    FOCUS_MAX_M: 1.0,
    VIDEO_FPS: 4,
    VIDEO_BITRATE: 2_000_000,
    VIDEO_CODEC: 'avc1.42001f',
    DEBUG: false
};

// 状態管理
const state = {
    elements: { video, canvas, ctx, playStopBtn, recordBtn, ... },
    camera: { track, focusSupported, mfSupported, focusMode, ... },
    animation: { isPlaying, currentFrame, intervalId, wasPausedByVisibility },
    recording: { isRecording, isConverting, capturedFrames },
    display: { gridMode }
};
```

**initCamera() の分割後:**
- `initCamera()` - メインオーケストレーター
- `startDrawLoop()` - 描画ループの開始
- `acquireCameraStream()` - 3段階フォールバックでカメラ取得
- `setupVideoElement()` - video要素の設定とストリーム接続
- `playVideoWithRetry()` - 再生開始（ユーザー操作リトライ付き）
- `scheduleVideoStateCheck()` - デバッグ用状態チェック

#### mp4-muxerライブラリの調査

MP4生成に使用している `mp4-muxer` の信頼性を調査：

| 項目 | 状態 |
|------|------|
| ライセンス | MIT |
| GitHub Stars | 592 ⭐ |
| 既知の脆弱性 | **なし**（Snykで確認済み） |
| メンテナンス | ⚠️ 非推奨（deprecated） |
| 後継 | Mediabunny |

**結論:** 現時点では安全に使用可能。脆弱性が見つかるか新機能が必要になった場合にMediabunnyへ移行を検討。

---

#### MP4録画ライブラリの変更（mp4-muxer → h264-mp4-encoder）

**問題:** ChromebookでMP4がダウンロードできず、PNGにフォールバックしてしまう

**原因:**
- 従来の方式はWebCodecs API（VideoEncoder）+ mp4-muxerを使用
- ChromebookではH.264のハードウェアエンコーダーが搭載されていないことが多い
- VideoEncoderがH.264をサポートしていないため、フォールバックが発生

**解決策:** h264-mp4-encoder に変更

| 項目 | 変更前（mp4-muxer） | 変更後（h264-mp4-encoder） |
|------|---------------------|----------------------------|
| エンコード方式 | WebCodecs API（ハードウェア依存） | WASM（ソフトウェアエンコード） |
| SharedArrayBuffer | 不要 | **不要** |
| Chromebook対応 | × | **○** |
| ライセンス | MIT | MIT + Public Domain + MPL 1.1 |

**h264-mp4-encoderの構成:**
- h264-mp4-encoder本体: MITライセンス
- minih264（H.264エンコーダー）: パブリックドメイン
- libmp4v2（MP4コンテナ）: MPL 1.1

**変更内容:**
1. CDNをmp4-muxerからh264-mp4-encoderに変更
2. createMP4FromFrames関数を完全に書き換え
3. CONFIG設定を調整（VIDEO_BITRATE, VIDEO_CODEC → VIDEO_QUALITY）

**コミット:** （後で追記）

---
