# Section 2c - データレイアウト変換

このセクションでは、タイルの専用ハードウェアを使用したオンザフライのデータレイアウト変換について説明します。この機能は**AIE-MLデバイスでのみ利用可能**です。Direct Memory Access（DMA）チャネルは、ローカルメモリモジュールとAXIストリーム相互接続の間でデータを移動し、プログラマブルなn次元アドレス生成スキームを可能にします。

## バッファディスクリプタ操作

MLIRでのバッファディスクリプタ操作（`AIE_DMABDOp`）は、移動するデータ、移動するデータの量、移動元のバッファ内の場所を指定します。この操作のPythonバインディング（`dma_bd`という名前）は、次のシグネチャを持ちます：

```python
def dma_bd(
    buffer,
    *,
    offset=None,
    len=None,
    dimensions=None,
    bd_id=None,
    next_bd_id=None,
    loc=None,
    ip=None,
)
```

このセクションでは、特にその`dimensions`パラメータについて詳しく説明します。以下の小さなコードスニペットを使用します。これは、`<MLIR_AIE_INSTALL_PATH>/python/aie/dialects/_aie_ops_gen.py`ファイルにあります。

## データレイアウト変換の形式

データレイアウト変換は、`size`（サイズ）と`stride`（ストライド）のペアのタプルとして指定されます。実際のハードウェア実装の制約により、最大次元数は、コンピュートタイルとシムタイルで3、メモリタイルで4に制限されています。以下の形式で指定します：

```c
[<size_2, stride_2>, <size_1, stride_1>, <size_0, stride_0>]
```

すべてのストライドは**要素幅の倍数**で表現されます。

> **注意**: **4Bデータ型の場合のみ、最内次元のストライドは設計上1でなければなりません。**

## ネストループモデル

基本的に、このデータレイアウト変換スキームはネストループとして見ることができ、ループボディ内の各反復でアクセス/保存する要素は、対応するバッファ内の連続した位置です。以下のCコードの例をご覧ください：

```c
int *buffer;
for(int i = 0; i < size_2; i++)
    for(int j = 0; j < size_1; j++)
        for(int k = 0; k < size_0; k++)
            // access/store element at
            // buffer[i * stride_2 + j * stride_1 + k * stride_0]
```

## 実践的な例

128要素のバッファから、偶数要素と奇数要素を交互に8要素のグループで4回（合計32要素）アクセスするMLIRでの例を見てみましょう：

```mlir
aie.dma_bd(%buf : memref<128xi32>, 0, 128, [<8, 16>, <2, 1>, <8, 2>])
```

実際にアクセスされるのは128要素中の64要素のみで、このアクセスパターンは以下のCコードで表現されます：

```c
int *buffer;
for(int i = 0; i < 8; i++)          // size_2
    for(int j = 0; j < 2; j++)      // size_1
        for(int k = 0; k < 8; k++)  // size_0
            // access/store element at
            // buffer[i * 16 + j * 1 + k * 2]
            // = buffer[16 * i + j + 2 * k]
```

## Object FIFOでのデータレイアウト変換

Object FIFOを使用してデータ移動を表現する場合、データレイアウト変換はMLIRにおける`object_fifo`クラスコンストラクタに渡すことができます。以下のクラスシグネチャをご覧ください：

```python
class object_fifo:
    def __init__(
        self,
        name,
        producerTile,
        consumerTiles,
        depth,
        datatype,
        dimensionsToStream=None,
        dimensionsFromStreamPerConsumer=None,
    ):
```

`dimensionsToStream`と`dimensionsFromStreamPerConsumer`は、それぞれプロデューサとコンシューマのDMAに対してデータレイアウト変換を指定します。

### 例

`<4x8xi8>`データ型のオブジェクトを持つObject FIFOを考えます。以下の例では、プロデューサがこのオブジェクトをストリームに送信する際、偶数行の最初の2要素のみを選択します。コンシューマ側では、ストリームからこれらの要素を取得し、メモリにデータとして保存します：

```python
A = tile(1, 1)
B = tile(1, 3)
of0 = object_fifo("objfifo0", A, B, 3, np.ndarray[(4, 8), np.dtype[np.int8]],
                  [(2, 16), (3, 2)])
```

`[(2, 16), (3, 2)]`の変換を対応するCコードで表現すると：

```c
int8_t *buffer;  // 4x8 = 32 elements
for(int i = 0; i < 2; i++)      // size_1: 2回の反復
    for(int j = 0; j < 3; j++)  // size_0: 3回の反復
        // access/store element at
        // buffer[i * 16 + j * 2]
```

これにより、要素インデックス0, 2, 4（1行目）と16, 18, 20（3行目）にアクセスします。

<img height="300" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/DataLayoutTransformation.svg">

## ランタイムシーケンスでのデータレイアウト変換

[Section 2d](../section-2d/README.md)でランタイムシーケンスプログラミングについて詳しく学びますが、ここで重要なのは、ランタイムシーケンス操作（`fill()`や`drain()`など）は、オプションで`tap`を入力として受け取り、外部メモリとの間のアクセスパターンをオンザフライで変更できるということです。IRONでのAI Engine用のテンソルアクセスパターンは、[こちら](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/tiling_exploration/README.md)で紹介されている`taplib`ライブラリによって提供されています。

データレイアウト変換を使用した注目すべき実装例については、[programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)ディレクトリ（例えば[matrix_vector_multiplication](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/matrix_vector/)や[matrix_multiplication_whole_array](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/whole_array/)）や、[dma_transpose](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/dma_transpose/)、[row_wise_bias_add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/row_wise_bias_add/)などのプログラミング例を参照してください。

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2c)を参照してください。
