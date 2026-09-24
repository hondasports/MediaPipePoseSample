# ゴルフ姿勢解析 設計ドキュメント

## 1. 概要

本プロジェクトでは、Android端末上で **MediaPipe Pose Landmarker** を利用し、ゴルフスイングなどの全身動作を解析する。

対象入力は以下の2種類とする。

- **ライブ解析**: CameraXから取得したカメラ映像をリアルタイム解析する
- **動画解析**: 端末内の動画ファイルを読み込み、時系列で姿勢を解析する

MediaPipe Pose Landmarkerから得られる33個の身体ランドマークを利用し、関節角度、身体の傾き、左右差、姿勢変化を算出する。

---

## 2. ゴール

### 2.1 MVP

MVPでは以下を実現する。

- Android端末のカメラ映像を表示する
- MediaPipe Pose Landmarkerで1人の姿勢を推定する
- 推定した骨格を映像上へオーバーレイ表示する
- 動画ファイルを選択し、姿勢解析できる
- フレームごとの主要ランドマークを取得する
- 以下の代表的な角度を算出する
  - 左右の肘角度
  - 左右の膝角度
  - 左右の股関節角度
  - 肩ラインの傾き
  - 腰ラインの傾き
  - 上体前傾
- timestampと姿勢データを対応付ける
- 推論FPS・処理時間を表示できる

### 2.2 将来拡張

- スイングフェーズ自動判定
  - Address
  - Takeaway
  - Top
  - Downswing
  - Impact
  - Follow-through
  - Finish
- 基準フォームとの比較
- 左右差の可視化
- 時系列グラフ
- スイングごとの履歴保存
- 複数スイング比較
- クラブ軌道解析
- ボール検出・弾道解析
- 音声フィードバック

---

## 3. MediaPipe Pose Landmarkerを採用する理由

MediaPipe Pose Landmarkerを採用する。

主な理由は以下。

- Android上でオンデバイス推論できる
- 33個の身体ランドマークを取得できる
- normalized landmarksだけでなくworld landmarksも取得できる
- IMAGE / VIDEO / LIVE_STREAMの各モードを利用できる
- トラッキングを利用して動画・ライブ入力を効率的に処理できる
- Lite / Full / Heavyモデルを用途に応じて選択できる
- 推論結果をアプリ外へ送信せずローカルで完結できる

ゴルフ用途では単純な骨格描画だけではなく、関節角度や身体回転の時系列解析が必要になるため、ML Kit Pose DetectionではなくMediaPipe Pose Landmarkerを標準とする。

---

## 4. モデル選択

MediaPipe BlazePose GHUM系には以下のモデルがある。

| モデル | 特徴 | 本プロジェクトでの用途 |
| --- | --- | --- |
| Lite | 高速・軽量 | 低性能端末、ライブ処理のフォールバック |
| Full | 速度と精度のバランス | **標準モデル** |
| Heavy | 高精度・高負荷 | 録画動画の詳細解析、比較検証 |

### 4.1 初期方針

最初は **Fullモデル** を標準とする。

理由:

- ゴルフスイングは腕・肩・腰・膝が高速に動く
- Liteでは端末負荷は低いが、フォーム解析では精度を優先したい
- Heavyはライブ用途では負荷が高い
- Fullはライブと動画の両方を1モデルで検証しやすい

端末性能や発熱を確認し、ライブ時のみLiteへ切り替えられる構成とする。

---

## 5. 全体アーキテクチャ

```text
                    ┌───────────────────────────┐
                    │        Android App        │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
            ┌───────────────┐           ┌───────────────┐
            │ Live Camera   │           │ Video File    │
            │ CameraX       │           │ Picker        │
            └───────┬───────┘           └───────┬───────┘
                    │                           │
                    ▼                           ▼
            ImageProxy / Frame           Decoded Frame
                    │                           │
                    └─────────────┬─────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ MediaPipe Pose Landmarker│
                    └─────────────┬─────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  ▼                               ▼
        Normalized Landmarks              World Landmarks
                  │                               │
                  └───────────────┬───────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │      Pose Analyzer        │
                    │                           │
                    │ - joint angles            │
                    │ - shoulder tilt           │
                    │ - hip tilt                │
                    │ - torso lean              │
                    │ - temporal features       │
                    └─────────────┬─────────────┘
                                  │
                       ┌──────────┴──────────┐
                       ▼                     ▼
                Overlay Renderer       Analysis Data
```

---

## 6. ライブ解析

### 6.1 処理フロー

```text
CameraX
  ↓
ImageAnalysis
  ↓
ImageProxy
  ↓
MPImage
  ↓
PoseLandmarker (LIVE_STREAM)
  ↓
resultCallback
  ↓
PoseAnalyzer
  ↓
Overlay
```

MediaPipeの `LIVE_STREAM` モードを利用する。

ライブモードでは推論が非同期で行われるため、カメラプレビューの描画を推論完了待ちでブロックしない。

処理が追いつかない場合に全フレームの解析を保証する必要はない。
姿勢解析では「最新の姿勢を低遅延で表示すること」を優先する。

### 6.2 推論FPS

カメラ映像とPose推論のFPSは分離する。

例:

```text
Camera Preview : 60 fps
Pose Inference : 15 - 30 fps
Overlay        : latest result
```

60fpsの全フレームにPose推論を実行する設計にはしない。

理由:

- CPU/GPU負荷の削減
- 発熱の抑制
- バッテリー消費の削減
- UIレイテンシの低減

ゴルフスイングの高速部分については、端末ごとに15 / 30fpsを比較し、必要に応じて推論間隔を調整する。

---

## 7. 動画解析

### 7.1 処理フロー

```text
Video File
  ↓
Video Decoder
  ↓
Frame + timestamp
  ↓
PoseLandmarker (VIDEO)
  ↓
PoseResult
  ↓
PoseAnalyzer
  ↓
Time-series Result
```

動画解析では `VIDEO` モードを使用する。

各フレームには単調増加するtimestampを付与し、姿勢データも同じtimestampで管理する。

### 7.2 動画FPSと推論FPS

録画された動画のフレームレートと推論フレームレートは分離する。

例:

```text
Recorded Video : 60 fps
Pose Analysis  : 30 fps
```

最初は30fps解析を基本とし、端末性能・精度・処理時間を測定する。

スロー動画や120fps / 240fps動画に対応する場合も、全フレームに推論するのではなく、必要な解析周期へサンプリングする。

---

## 8. 取得するランドマーク

Pose Landmarkerは33個のランドマークを返す。

ゴルフ解析では特に以下を使用する。

### 上半身

- LEFT_SHOULDER
- RIGHT_SHOULDER
- LEFT_ELBOW
- RIGHT_ELBOW
- LEFT_WRIST
- RIGHT_WRIST
- LEFT_INDEX
- RIGHT_INDEX

### 体幹

- LEFT_HIP
- RIGHT_HIP

### 下半身

- LEFT_KNEE
- RIGHT_KNEE
- LEFT_ANKLE
- RIGHT_ANKLE
- LEFT_HEEL
- RIGHT_HEEL
- LEFT_FOOT_INDEX
- RIGHT_FOOT_INDEX

---

## 9. データモデル案

```kotlin
data class PoseFrame(
    val timestampMs: Long,
    val landmarks: List<PosePoint>,
    val worldLandmarks: List<PosePoint>,
    val metrics: PoseMetrics,
)

data class PosePoint(
    val index: Int,
    val x: Float,
    val y: Float,
    val z: Float,
    val visibility: Float?,
    val presence: Float?,
)

data class PoseMetrics(
    val leftElbowAngle: Float?,
    val rightElbowAngle: Float?,
    val leftKneeAngle: Float?,
    val rightKneeAngle: Float?,
    val leftHipAngle: Float?,
    val rightHipAngle: Float?,
    val shoulderTilt: Float?,
    val hipTilt: Float?,
    val torsoLean: Float?,
)
```

MediaPipe固有の型をUIや解析ロジックへ直接広げず、アプリ内部の型へ変換する。

これにより、以下を容易にする。

- MediaPipeバージョン変更
- テストデータ作成
- 保存形式変更
- 将来的な推定モデル差し替え

---

## 10. 関節角度

3点 `A-B-C` から、Bを頂点とする角度を求める。

```text
A
 \
  \
   B -------- C
```

ベクトル:

```text
BA = A - B
BC = C - B
```

角度:

```text
θ = acos((BA · BC) / (|BA| |BC|))
```

### 例

左肘:

```text
LEFT_SHOULDER
      ↓
LEFT_ELBOW
      ↓
LEFT_WRIST
```

左膝:

```text
LEFT_HIP
   ↓
LEFT_KNEE
   ↓
LEFT_ANKLE
```

### 2D / 3D

UI表示用の単純な角度はnormalized landmarksから算出できる。

フォーム解析ではworld landmarksを利用した3D角度も比較する。

ただし単眼カメラから推定される3D座標はモーションキャプチャ計測値と同等ではないため、絶対値ではなく同一条件での相対比較を重視する。

---

## 11. ゴルフ向け解析項目

### 11.1 Address

- 肩幅
- 足幅
- 膝角度
- 上体前傾
- 肩ライン
- 腰ライン
- 左右荷重の代替指標

### 11.2 Backswing / Top

- 左右肩位置
- 腰回転
- 肩と腰の相対回転
- 左腕角度
- 右肘角度
- 頭部位置変化
- 膝角度変化

### 11.3 Downswing / Impact

- 腰の回転開始タイミング
- 肩の回転
- 左右手首位置
- 左膝伸展
- 頭部位置変化
- 肩・腰の回転差

### 11.4 Finish

- 体幹角度
- 左右肩高さ
- 右足位置
- 左脚の安定性
- 重心移動の代替指標

---

## 12. スイングフェーズ判定

MVPではフェーズ判定を必須としない。

将来的にはランドマークの時系列特徴から判定する。

候補:

1. ルールベース
2. 時系列特徴量 + 閾値
3. TFLiteによる動作分類モデル

初期はルールベースを推奨する。

例:

```text
Address
  ↓
手首の上方向移動開始
  ↓
Takeaway
  ↓
手首高さ最大
  ↓
Top
  ↓
手首下降
  ↓
Downswing
  ↓
手首が腰付近を高速通過
  ↓
Impact candidate
  ↓
Finish
```

実データを集めてからML分類へ移行する。

---

## 13. カメラ方向

ゴルフ解析では撮影方向によって取得しやすい情報が異なる。

### Down-the-line

飛球線後方から撮影。

取得しやすいもの:

- 前傾維持
- スイングプレーンの近似
- 頭部移動
- 膝角度
- 手元位置

### Face-on

正面から撮影。

取得しやすいもの:

- 左右への体重移動の代替指標
- スウェー
- 肩・腰の傾き
- 左右膝
- 頭部の左右移動

初期実装では撮影方向をユーザーに選択させる設計を検討する。

```text
AnalysisMode
- FACE_ON
- DOWN_THE_LINE
```

同じ評価式を両撮影方向へ適用しない。

---

## 14. Overlay

映像上に以下を表示する。

- 33ランドマーク
- 骨格接続線
- 主要関節角度
- shoulder line
- hip line
- torso line
- 推論FPS
- 推論時間
- confidence / visibilityが低いポイントの警告

ライブモードでは最新のPose結果を保持し、CameraX Preview上へ描画する。

---

## 15. 信頼度

各ランドマークのvisibility / presenceを考慮する。

低信頼度ランドマークから算出した角度は解析結果として採用しない。

例:

```kotlin
if (shoulder.visibility < threshold ||
    elbow.visibility < threshold ||
    wrist.visibility < threshold
) {
    return null
}
```

閾値は固定値で決め打ちせず、実際のゴルフ動画で評価する。

---

## 16. スムージング

Pose結果はフレームごとに微小な揺れが発生する。

解析用にはスムージング層を設ける。

候補:

- Moving Average
- Exponential Moving Average
- One Euro Filter
- Kalman Filter

初期実装ではEMAまたはOne Euro Filterを候補とする。

ただし、過度な平滑化はインパクト付近の高速動作を潰すため、表示用と解析用でパラメータを分離できる設計とする。

---

## 17. 推奨パッケージ構成

```text
app/
└── src/main/java/.../
    ├── camera/
    │   ├── CameraController.kt
    │   └── CameraFrameAnalyzer.kt
    │
    ├── video/
    │   ├── VideoFrameSource.kt
    │   └── VideoAnalyzer.kt
    │
    ├── pose/
    │   ├── PoseLandmarkerController.kt
    │   ├── PoseMapper.kt
    │   └── model/
    │       ├── PoseFrame.kt
    │       ├── PosePoint.kt
    │       └── PoseMetrics.kt
    │
    ├── analysis/
    │   ├── PoseAnalyzer.kt
    │   ├── JointAngleCalculator.kt
    │   ├── GolfSwingAnalyzer.kt
    │   ├── PoseSmoother.kt
    │   └── SwingPhaseDetector.kt
    │
    ├── overlay/
    │   └── PoseOverlay.kt
    │
    └── ui/
        ├── live/
        └── video/
```

重要なのは、

```text
MediaPipe
    ↓
PoseMapper
    ↓
Domain Model
    ↓
GolfSwingAnalyzer
```

と分離すること。

`GolfSwingAnalyzer` がMediaPipe APIへ直接依存しないようにする。

---

## 18. MediaPipe設定案

初期値:

```text
runningMode:
  Live  -> LIVE_STREAM
  Video -> VIDEO

numPoses:
  1

minPoseDetectionConfidence:
  0.5

minPosePresenceConfidence:
  0.5

minTrackingConfidence:
  0.5

outputSegmentationMasks:
  false
```

最初は公式デフォルト付近から開始し、実際のゴルフ映像を使って調整する。

セグメンテーションマスクはMVPでは不要。

---

## 19. パフォーマンス計測

必ず実機で以下を記録する。

- Camera FPS
- Pose inference FPS
- 平均推論時間
- P50 / P95推論時間
- CPU使用率
- メモリ使用量
- 発熱
- 5分 / 10分連続動作時の性能低下

モデルごとに比較する。

```text
Device:
Model:
Input resolution:
Camera FPS:
Pose FPS:
Avg inference:
P95 inference:
Temperature:
```

ベンチマーク結果を見てLite / Full / Heavyを決定する。

---

## 20. 制約

### 20.1 高速動作

ゴルフスイング、とくにインパクト前後は非常に高速。

モーションブラーにより、

- 手首
- 肘
- 指
- クラブ

の検出精度が低下する可能性がある。

十分な明るさと短い露光時間が重要になる。

### 20.2 クラブ

Pose Landmarkerは人体姿勢推定モデルであり、ゴルフクラブそのものを追跡するモデルではない。

クラブヘッド・シャフト軌道を解析する場合は別モデルが必要。

候補:

- Object Detection
- カスタムTFLiteモデル
- 画像処理によるライン検出

### 20.3 単眼3D

world landmarksは有用だが、単一RGBカメラから推定された3D情報である。

以下の用途では注意する。

- 厳密な距離計測
- 重心位置の物理計測
- 関節の医療用途レベルの角度計測
- モーションキャプチャとの絶対値比較

本アプリでは主にフォーム比較・相対変化・時系列解析に使用する。

---

## 21. MVP実装順序

### Phase 1: Pose表示

- Androidプロジェクト作成
- MediaPipe Tasks Vision追加
- Pose Landmarker Fullモデル追加
- 静止画でPose推論
- 骨格描画

### Phase 2: ライブ

- CameraX Preview
- ImageAnalysis
- LIVE_STREAM Pose推論
- Overlay
- FPS表示

### Phase 3: 動画

- 動画選択
- フレームデコード
- VIDEO Pose推論
- timestamp同期
- 再生画面へのOverlay

### Phase 4: ゴルフ解析

- 関節角度
- shoulder / hip line
- torso lean
- 時系列保存
- グラフ表示

### Phase 5: Swing Phase

- Address
- Top
- Impact候補
- Finish

### Phase 6: 評価

- Lite / Full / Heavy比較
- Camera angle比較
- 実機ベンチマーク
- 実ゴルフ動画で精度確認

---

## 22. 最初に作る画面

### Home

```text
+-----------------------------+
| MediaPipe Pose Sample       |
|                             |
| [ Live Analysis ]           |
|                             |
| [ Analyze Video ]           |
+-----------------------------+
```

### Live Analysis

```text
+-----------------------------+
| Camera Preview              |
|                             |
|      Pose Overlay           |
|                             |
| Left knee : 145°            |
| Right knee: 138°            |
| Pose FPS  : 24              |
+-----------------------------+
```

### Video Analysis

```text
+-----------------------------+
| Video + Pose Overlay        |
|                             |
|                             |
| [---------●------------]    |
| 00:01.240                   |
|                             |
| Shoulder tilt: 8.2°         |
| Hip tilt     : 3.1°         |
+-----------------------------+
```

---

## 23. 参考資料

- MediaPipe Pose Landmarker
  - https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker
- MediaPipe Tasks
  - https://ai.google.dev/edge/mediapipe/solutions/tasks
- BlazePose GHUM 3D Model Card
  - https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf

---

## 24. 方針まとめ

本サンプルでは以下を基本方針とする。

- 姿勢推定ライブラリは **MediaPipe Pose Landmarker**
- 対象人物はまず1人
- 標準モデルは **Full**
- ライブは **LIVE_STREAM**
- 動画は **VIDEO**
- カメラFPSと推論FPSを分離
- MediaPipe固有型とゴルフ解析ドメインを分離
- まずランドマーク・関節角度を正しく取得する
- スイングフェーズ判定は実データ取得後に追加する
- クラブ軌道解析はPose Landmarkerとは別機能として扱う
