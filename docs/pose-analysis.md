# 姿勢解析 設計ドキュメント

## 1. 概要

本プロジェクトでは、Android端末上で **MediaPipe Pose Landmarker** を利用し、ライブ映像または動画から人体姿勢を解析する。

特定の競技や運動に依存せず、姿勢解析の共通基盤を構築することを目的とする。

対象入力は以下の2種類とする。

- **ライブ解析**: CameraXから取得したカメラ映像をリアルタイム解析する
- **動画解析**: 端末内の動画ファイルを読み込み、時系列で姿勢を解析する

MediaPipe Pose Landmarkerから取得した身体ランドマークをアプリ内部の共通データモデルへ変換し、その後に関節角度・身体の傾き・左右差・移動量などを算出する。

---

## 2. 設計方針

姿勢解析を以下の4層に分離する。

```text
Input
  ↓
Pose Estimation
  ↓
Common Pose Analysis
  ↓
Application-specific Analysis
```

### Input

画像や映像をMediaPipeへ渡せる形式に変換する。

### Pose Estimation

MediaPipe Pose Landmarkerを利用して人体ランドマークを取得する。

### Common Pose Analysis

ランドマークから用途に依存しない姿勢指標を計算する。

### Application-specific Analysis

共通姿勢指標を利用して、特定の動作やフォームを評価する。

MediaPipe APIを用途固有のロジックへ直接露出させない。

---

## 3. MVP

MVPでは以下を実現する。

- Android端末のカメラ映像を表示する
- MediaPipe Pose Landmarkerで1人の姿勢を推定する
- 推定した骨格を映像上へオーバーレイ表示する
- 動画ファイルを選択し、姿勢解析できる
- フレームごとのランドマークを取得する
- 主要な関節角度を算出する
- 肩・腰・体幹の傾きを算出する
- timestampと姿勢データを対応付ける
- 推論FPS・処理時間を表示する

MVPでは、特定動作の良し悪し判定や動作分類は必須としない。

---

## 4. MediaPipe Pose Landmarker

姿勢推定には **MediaPipe Pose Landmarker** を利用する。

利用理由:

- Android上でオンデバイス推論できる
- 33個の身体ランドマークを取得できる
- normalized landmarksを取得できる
- world landmarksを取得できる
- IMAGE / VIDEO / LIVE_STREAMの各モードを利用できる
- 動画・ライブ入力ではトラッキングを利用できる
- Lite / Full / Heavyモデルを用途に応じて選択できる
- 推論結果を端末外へ送信せずローカルで処理できる

---

## 5. モデル選択

| モデル | 特徴 | 想定用途 |
| --- | --- | --- |
| Lite | 高速・軽量 | 低性能端末、ライブ処理 |
| Full | 速度と精度のバランス | **標準モデル** |
| Heavy | 高精度・高負荷 | 録画動画の詳細解析、比較検証 |

初期実装では **Fullモデル** を標準とする。

端末負荷、発熱、推論速度を計測し、ライブ時のみLiteへ切り替えられる構成を許容する。

Heavyはオフライン動画解析や精度比較用として扱う。

---

## 6. 全体アーキテクチャ

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
                    │        PoseMapper         │
                    └─────────────┬─────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │         PoseFrame         │
                    └─────────────┬─────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │       PoseAnalyzer        │
                    │ - joint angles            │
                    │ - alignment               │
                    │ - symmetry                │
                    │ - motion                  │
                    └─────────────┬─────────────┘
                                  │
                       ┌──────────┴──────────┐
                       ▼                     ▼
                Overlay Renderer      Domain Analyzer
```

---

## 7. ライブ解析

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
PoseMapper
  ↓
PoseAnalyzer
  ↓
Overlay
```

ライブ解析では `LIVE_STREAM` モードを利用する。

カメラプレビューとPose推論を分離し、推論完了待ちでUIをブロックしない。

カメラ映像とPose推論のFPSも分離する。

```text
Camera Preview : 60 fps
Pose Inference : 15 - 30 fps
Overlay        : latest result
```

全フレームにPose推論を実行することは前提にしない。

---

## 8. 動画解析

```text
Video File
  ↓
Video Decoder
  ↓
Frame + timestamp
  ↓
PoseLandmarker (VIDEO)
  ↓
PoseMapper
  ↓
PoseAnalyzer
  ↓
Time-series Result
```

動画解析では `VIDEO` モードを使用する。

各フレームには単調増加するtimestampを付与し、姿勢データも同じtimestampで管理する。

録画動画のFPSと推論FPSは分離する。

```text
Recorded Video : 60 fps
Pose Analysis  : 30 fps
```

高フレームレート動画でも、必要な解析周期へサンプリングする。

---

## 9. ランドマーク

Pose Landmarkerの33ランドマークを利用する。

主に以下を共通姿勢解析へ使用する。

### 頭部

- NOSE
- LEFT_EYE / RIGHT_EYE
- LEFT_EAR / RIGHT_EAR

### 上半身

- LEFT_SHOULDER / RIGHT_SHOULDER
- LEFT_ELBOW / RIGHT_ELBOW
- LEFT_WRIST / RIGHT_WRIST
- LEFT_INDEX / RIGHT_INDEX

### 体幹

- LEFT_HIP / RIGHT_HIP

### 下半身

- LEFT_KNEE / RIGHT_KNEE
- LEFT_ANKLE / RIGHT_ANKLE
- LEFT_HEEL / RIGHT_HEEL
- LEFT_FOOT_INDEX / RIGHT_FOOT_INDEX

---

## 10. 内部データモデル

MediaPipe固有型を解析ロジックへ直接渡さず、アプリ内部の型へ変換する。

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

この分離により以下を容易にする。

- MediaPipeバージョン変更
- テストデータ作成
- 保存形式変更
- 推定モデル差し替え
- 用途固有Analyzerの追加

---

## 11. 共通姿勢解析

### 11.1 関節角度

3点 `A-B-C` から、Bを頂点とする角度を求める。

```text
BA = A - B
BC = C - B

θ = acos((BA · BC) / (|BA| |BC|))
```

例:

- 肩 - 肘 - 手首 → 肘角度
- 腰 - 膝 - 足首 → 膝角度
- 肩 - 腰 - 膝 → 股関節角度

### 11.2 アライメント

以下を共通指標として扱う。

- 肩ラインの傾き
- 腰ラインの傾き
- 体幹の傾き
- 頭部位置
- 左右肩の高さ差
- 左右腰の高さ差

### 11.3 左右差

左右に対応するランドマーク・角度を比較する。

- 左右肘角度差
- 左右膝角度差
- 左右肩高さ差
- 左右腰高さ差

### 11.4 動き

timestamp付きPoseFrameから時系列特徴を計算する。

- 位置変化
- 速度
- 角速度
- 角度変化
- 移動方向

---

## 12. 2D / 3D

UI表示用の単純な角度はnormalized landmarksから算出できる。

より立体的なフォーム解析ではworld landmarksを利用した3D角度も利用する。

ただし、world landmarksは単一RGBカメラから推定された3D情報であり、専用モーションキャプチャによる計測値と同等ではない。

絶対値よりも同一条件での相対比較を重視する。

---

## 13. 信頼度

各ランドマークのvisibility / presenceを考慮する。

低信頼度ランドマークから算出した値は解析結果として採用しない。

```kotlin
if (shoulder.visibility < threshold ||
    elbow.visibility < threshold ||
    wrist.visibility < threshold
) {
    return null
}
```

閾値は固定値で決め打ちせず、実データで評価して調整する。

---

## 14. スムージング

Pose結果にはフレームごとの微小な揺れが発生するため、解析用にスムージング層を設ける。

候補:

- Moving Average
- Exponential Moving Average
- One Euro Filter
- Kalman Filter

初期実装ではEMAまたはOne Euro Filterを候補とする。

過度な平滑化は高速動作を潰すため、表示用と解析用でパラメータを分離できる構成とする。

---

## 15. Overlay

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

## 16. パッケージ構成案

```text
app/
└── src/main/java/.../
    ├── camera/
    │   ├── CameraController.kt
    │   └── CameraFrameAnalyzer.kt
    ├── video/
    │   ├── VideoFrameSource.kt
    │   └── VideoAnalyzer.kt
    ├── pose/
    │   ├── PoseLandmarkerController.kt
    │   ├── PoseMapper.kt
    │   └── model/
    │       ├── PoseFrame.kt
    │       ├── PosePoint.kt
    │       └── PoseMetrics.kt
    ├── analysis/
    │   ├── PoseAnalyzer.kt
    │   ├── JointAngleCalculator.kt
    │   ├── BodyAlignmentAnalyzer.kt
    │   ├── MotionAnalyzer.kt
    │   └── PoseSmoother.kt
    ├── overlay/
    │   └── PoseOverlay.kt
    └── ui/
        ├── live/
        └── video/
```

依存方向は以下とする。

```text
MediaPipe
    ↓
PoseMapper
    ↓
PoseFrame
    ↓
PoseAnalyzer
    ↓
Application-specific Analyzer
```

共通解析層はMediaPipe固有APIや特定用途のルールへ依存しない。

---

## 17. MediaPipe設定案

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

最初は公式デフォルト付近から開始し、実データを使って調整する。

---

## 18. パフォーマンス計測

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

---

## 19. 制約

### 19.1 高速動作

高速な動作ではモーションブラーによりランドマーク精度が低下する可能性がある。

必要に応じて以下を考慮する。

- 十分な明るさ
- 短い露光時間
- 高フレームレート撮影
- 推論頻度の調整

### 19.2 遮蔽

身体の一部が他の部位や物体に隠れると、visibilityが低下する可能性がある。

### 19.3 単眼3D

world landmarksは有用だが、単一RGBカメラから推定された3D情報である。

以下の用途では注意する。

- 厳密な距離計測
- 重心位置の物理計測
- 医療用途レベルの関節角度計測
- 専用モーションキャプチャとの絶対値比較

---

## 20. MVP実装順序

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

### Phase 4: 共通姿勢解析

- 関節角度
- shoulder / hip line
- torso lean
- 左右差
- 時系列データ
- 速度・角度変化

### Phase 5: 評価

- Lite / Full / Heavy比較
- 実機ベンチマーク
- 動画とライブの精度比較

---

## 21. 画面案

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
| [---------●------------]    |
| 00:01.240                   |
|                             |
| Shoulder tilt: 8.2°         |
| Hip tilt     : 3.1°         |
+-----------------------------+
```

---

## 22. 参考資料

- MediaPipe Pose Landmarker
  - https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker
- MediaPipe Tasks
  - https://ai.google.dev/edge/mediapipe/solutions/tasks
- BlazePose GHUM 3D Model Card
  - https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf

---

## 23. 方針まとめ

- 姿勢推定ライブラリは **MediaPipe Pose Landmarker**
- 対象人物はまず1人
- 標準モデルは **Full**
- ライブは **LIVE_STREAM**
- 動画は **VIDEO**
- カメラFPSと推論FPSを分離する
- MediaPipe固有型を共通解析層へ持ち込まない
- 姿勢推定・共通解析・用途固有解析を分離する
- まずランドマーク・関節角度・アライメント・時系列解析を正しく実装する
