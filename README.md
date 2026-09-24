# MediaPipePoseSample

Androidで **MediaPipe Pose Landmarker** を利用し、ゴルフスイングなどの姿勢をリアルタイムまたは動画から解析するためのサンプルプロジェクトです。

## 目的

- カメラ映像からリアルタイムに人体の骨格を推定する
- 端末内の動画を読み込み、フレーム単位で姿勢を解析する
- ゴルフスイングに必要な関節角度・身体の傾き・左右差などを算出する
- 推定結果を映像上へオーバーレイ表示する
- 将来的にスイングフェーズ判定やフォーム比較へ拡張できる構成にする

## 採用ライブラリ

- MediaPipe Tasks Vision
- Pose Landmarker
- Android CameraX（ライブ映像）
- Media3 / MediaMetadataRetriever 等（動画入力。実装時に方式を確定）

Pose Landmarkerは33個の身体ランドマークを取得でき、画像座標に加えてworld landmarksも利用できます。

## ドキュメント

- [ゴルフ姿勢解析 設計ドキュメント](docs/golf-pose-analysis.md)

## 想定する最初のスコープ

1. 1人のゴルファーを対象とする
2. ライブカメラ解析
3. 動画ファイル解析
4. 骨格オーバーレイ
5. 主要関節角度の計算
6. スイング中の時系列データ保存
7. アドレス・トップ・インパクト・フィニッシュ等のフェーズ判定は後続フェーズで追加

## 参考

- MediaPipe Pose Landmarker:
  https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker
- MediaPipe Tasks:
  https://ai.google.dev/edge/mediapipe/solutions/tasks
