# Section 5 - ベクトル設計の例

[プログラミング例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples)は、Ryzen™ AIのAI EngineとNPU配列の独自機能をさらに説明するための多数のサンプル設計です。

## 最もシンプルな例

#### Passthrough

[passthrough](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/passthrough_kernel/)例は、最もシンプルな「入門」例です。ベクトル化されたロードとストアを使用して、入力から出力へ4096バイトをコピーします。この設計例は、他の例でも簡単に再現できる典型的なプロジェクト構成を示しています。ここで重要なファイルは実質4つだけです。

1. [`passthrough_kernel.py`](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_examples/basic/passthrough_kernel/passthrough_kernel.py) - 外部メモリに接続されたシムタイルと、コピーを実行する単一のAIEコアを含むAIE構造設計。[セクション2](../section-2/index.html)で説明されているObject FIFOの簡単な使用例も示しています。
2. [`passthrough.cc`](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/aie_kernels/generic/passThrough.cc) - ベクトル化されたコピー操作を実行するC++ファイル。
3. [`test.cpp`](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_examples/basic/passthrough_kernel/test.cpp) または [`test.py`](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_examples/basic/passthrough_kernel/test.py) - 設計を実行し、CPUリファレンスと比較するためのC++またはPythonメインアプリケーション。
4. [`Makefile`](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_examples/basic/passthrough_kernel/Makefile) - 様々なアーティファクトのビルドプロセスを文書化（および実装）するMakefile。

[passthrough DMAs](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/passthrough_dmas/)例は、コアを使わずにループバックを実行してコピーを行う別の方法を示しています。

## 基本設計

| 設計名 | データ型 | 説明 |
|-|-|-|
| [Vector Scalar Add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_add/) | i32 | ベクトルの各要素に1を加算 |
| [Vector Scalar Mul](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/) | i32 | ベクトルにスケールファクタを乗算 |
| [Vector Vector Add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_vector_add/) | i32 | 2つのベクトルを加算 |
| [Vector Vector Modulo](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_vector_modulo/) | i32 | ベクトル同士の剰余演算 |
| [Vector Vector Multiply](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_vector_mul/) | i32 | 2つのベクトルの要素ごとの乗算 |
| [Vector Reduce Add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_reduce_add/) | bfloat16 | ベクトルの全要素の合計値を返す |
| [Vector Reduce Max](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_reduce_max/) | bfloat16 | ベクトルの全要素の最大値を返す |
| [Vector Reduce Min](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_reduce_min/) | bfloat16 | ベクトルの全要素の最小値を返す |
| [Vector Exp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_exp/) | bfloat16 | 入力のe<sup>x</sup>を表すベクトルを返す |
| [DMA Transpose](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/dma_transpose/) | i32 | `npu_dma_memcpy_nd`を使用してShim DMAで行列を転置 |
| [Matrix Scalar Add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_scalar_add/) | i32 | 行列とスカラーの乗算 |
| [Single core GEMM](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/single_core/) | bfloat16 | 単一コアの行列-行列乗算 |
| [Multi core GEMM](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/whole_array/) | bfloat16 | オペランドブロードキャストを使用した16個のAIEによる行列-行列乗算。シンプルな「その場で累積」戦略を使用 |
| [GEMV](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/matrix_vector/) | bfloat16 | ベクトルを返すベクトル-行列乗算 |

## 機械学習カーネル

| 設計名 | データ型 | 説明 |
|-|-|-|
| [Eltwise Add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/eltwise_add/) | bfloat16 | 2つのベクトルの要素ごとの加算 |
| [Eltwise Mul](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/eltwise_mul/) | i32 | 2つのベクトルの要素ごとの乗算 |
| [ReLU](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/relu/) | bfloat16 | ベクトルに対する正規化線形ユニット（ReLU）活性化関数 |
| [Softmax](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/softmax/) | bfloat16 | 行列に対するSoftmax演算 |
| [Conv2D](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/conv2d/) | i8 | CNN用の単一コア2次元畳み込み |
| [Conv2D+ReLU](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/conv2d_fused_relu/) | i8 | ベクトルレジスタレベルでReLUが融合されたConv2D |

## 演習問題

1. [passthrough](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/passthrough_kernel/)設計を変更して、より多く（または少なく）データをコピーできますか？
   <details>
   <summary>答えを見る</summary>
   Makefileを確認してください。in1_sizeとout_sizeを変更します。
   </details>

2. [Vector Exp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_exp/)例の[test.cpp](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_examples/basic/vector_exp/test.cpp)のテストベンチを見てください。データ型とテストベクトルのサイズに注目してください。何に気づきますか？
   <details>
   <summary>答えを見る</summary>
   65536個の値、つまり2^16個をテストしています。したがって、すべての可能なbfloat16値を近似を通してテストしています。
   </details>

3. [ReLU](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/relu/)における通信対計算比は何ですか？
   <details>
   <summary>答えを見る</summary>
   トレースによると約6です。これが、Conv2DやGEMMとのカーネル融合が機械学習において有効な理由です。
   </details>

4. **難問** どの基本例が[Softmax](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/ml/softmax/)のコンポーネントになっていますか？
   <details>
   <summary>答えを見る</summary>
   Vector Exp
   </details>
