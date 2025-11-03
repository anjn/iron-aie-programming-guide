# Section 6 - 大規模設計の例

Ryzen™ AIのAI EngineとNPU配列の独自機能をさらに説明するための[多数の設計例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)が利用可能です。このセクションでは、ビジョンと機械学習のユースケース向けの、より複雑なアプリケーション設計を紹介します。特に、Ryzen™ AI上のResNet実装について説明します。

## ビジョンカーネル

| 設計名 | データ型 | 説明 |
|-|-|-|
| [Vision Passthrough](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/vision/vision_passthrough/) | i8 | 1つの`passThrough`カーネルのみを含むシンプルなパイプライン。このパイプラインは主に、グレースケール画像をコピーするためのデータ移動が正しく動作するかをテストすることを目的としています。 |
| [Color Detect](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/vision/color_detect/) | i32 | マルチカーネル、マルチコアパイプラインで、RGBA画像内の色を検出します。 |
| [Edge Detect](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/vision/edge_detect/) | i32 | マルチカーネル、マルチコアパイプラインで、画像内のエッジを検出し、元の画像に検出結果を重ね合わせます。 |
| [Color Threshold](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/vision/color_threshold/) | i32 | RGBA画像のカラーしきい値処理のマルチコア、データ並列実装。 |

## 機械学習設計

| 設計名 | データ型 | 説明 |
|-|-|-|
| [bottleneck](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/bottleneck/) | ui8 | ボトルネック残差ブロックは、1x1、3x3、1x1のフィルタサイズを使用する3つの畳み込みを利用する残差ブロックの変種です。この実装は、複数のカーネルの融合とデータフロー最適化を特徴とし、AI Engineの独自のアーキテクチャ機能を強調しています。 |
| [resnet](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/resnet/) | ui8 | conv2_x層をオフロードしたResNet。この実装は、複数のNPU列にわたる複数のボトルネックブロックの深さ優先実装を特徴としています。 |

## 演習問題

1. [bottleneck](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/bottleneck/)設計では、何種類の融合計算が観察できますか？ <img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/mlir_tutorials/images/answer1.jpg" title="2. Convolution fused with ReLU and Convolution fused with element-wise addition. Please note that fusing adjacent convolution and batch norm layers is another inference-time optimization, which our implementation can handle." height=25>

2. [bottleneck](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/bottleneck/)設計でデータフローアプローチに従う場合、3x3畳み込み演算が計算を進めるために1x1畳み込みコアからいくつの要素を必要としますか？ <img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/mlir_tutorials/images/answer1.jpg" title="3. This allows for the necessary neighborhood information required by the convolutional kernel to be available for processing." height=25>

3. 入力次元が32x32x256のボトルネックブロックがあるとします。1x1畳み込み層を通過した後、出力次元は32x32x64になります。その後の3x3畳み込み層（ストライド1、パディングなし、出力チャネル64）を通過した後の出力次元はどうなりますか？ <img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/mlir_tutorials/images/answer1.jpg" title="30×30×64. Without padding, the spatial dimensions would shrink by two pixels in each dimension due to the 3x3 convolution operation." height=25>

-----

[[前へ - Section 5](../section-5/index.html)] [[トップ](../index.html)]
