# MediaPipePoseSample

Androidで **MediaPipe Pose Landmarker** を利用し、ライブ映像または動画から人体姿勢を解析するためのサンプルプロジェクトです。

## 目的

- カメラ映像からリアルタイムに人体の骨格を推定する
- 端末内の動画を読み込み、時系列で姿勢を解析する
- 関節角度・身体の傾き・左右差・移動量などの汎用的な姿勢指標を算出する
- 推定結果を映像上へオーバーレイ表示する
- 姿勢推定と用途固有の評価ロジックを分離し、さまざまな動作解析へ拡張できる構成にする

## 採用ライブラリ

- MediaPipe Tasks Vision
- Pose Landmarker
- Android CameraX（ライブ映像）
- Media3 / MediaMetadataRetriever 等（動画入力。実装時に方式を確定）

Pose Landmarkerから33個の身体ランドマークを取得し、normalized landmarks と world landmarks を姿勢解析に利用します。

## ドキュメント

- [姿勢解析 設計ドキュメント](docs/pose-analysis.md)

## 想定する最初のスコープ

1. 1人を対象とした姿勢推定
2. ライブカメラ解析
3. 動画ファイル解析
4. 骨格オーバーレイ
5. 主要関節角度の計算
6. 肩・腰・体幹などのアライメント解析
7. timestamp付き時系列データの生成
8. 用途固有の動作判定は後段のAnalyzerとして追加

## 基本方針

```text
Input
  ↓
Pose Estimation
  ↓
Common Pose Analysis
  ↓
Application-specific Analysis
```

MediaPipeは人体ランドマークの推定に集中させ、動作ごとの評価ロジックは独立した層に実装します。

## 参考

- MediaPipe Pose Landmarker:
  https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker
- MediaPipe Tasks:
  https://ai.google.dev/edge/mediapipe/solutions/tasks
