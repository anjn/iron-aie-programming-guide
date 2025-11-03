# Section 2e - マルチコアプログラミング

このセクションでは、単一コア設計をマルチコア設計に変換する方法を実演します。高レベルIRONと明示的配置レベルIRONの両方について説明します。

## 高レベルIRON

### 初期設定とデータ移動

最初のシンプルな設計（[aie2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2e/aie2.py)）では、`<48xi32>`データ型のオブジェクトを持つ4つのObject FIFOを使用します：外部メモリとメモリタイル間の入力用に1つ(`of_in`)、メモリタイルとWorker間の入力用に1つ(`of_in1`)、Worker実行出力用に1つ(`of_out1`)、出力をメモリタイルから外部メモリに戻すために1つ(`of_out`)です。Workerのタスクは、入力を取得し、エントリを1ずつ増加させ、その結果を出力に格納し、両方を解放することです。

**単一Worker設計のデータ移動:**

```python
data_size = 48

# テンソル型の定義
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# 入力データ移動
of_in = ObjectFifo(data_ty, name="in")
of_in1 = of_in.cons().forward(obj_type=data_ty, name="in1")

# 出力データ移動
of_out1 = ObjectFifo(data_ty, name="out1")
of_out = of_out1.cons().forward(obj_type=data_ty, name="out")
```

スケールアウトした設計（[aie2_multi.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2e/aie2_multi.py)）では、単一のメモリタイルを維持しますが、3つのWorkerに拡張します。Object FIFOオブジェクトのデータ型は`<16xi32>`に変更され、複数のWorker間でデータを分散および結合するためのObject FIFOリンク（[分散と結合パターンを参照](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2b/03_Implicit_Copy/README.md)）が追加されました：

**マルチWorker設計のデータ移動:**

```python
n_workers = 3
data_size = 48
tile_size = data_size // 3

# テンソル型の定義
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]
tile_ty = np.ndarray[(tile_size,), np.dtype[np.int32]]

# 入力データ移動
of_offsets = [tile_size * worker for worker in range(n_workers)]

of_in = ObjectFifo(data_ty, name="in")
of_ins = (
    of_in
    .cons()
    .split(
        of_offsets,
        obj_types=[tile_ty] * n_workers,
        names=[f"in{worker}" for worker in range(n_workers)],
    )
)

# 出力データ移動
of_out = ObjectFifo(data_ty, name="out")
of_outs = (
    of_out.prod().join(
        of_offsets,
        obj_types=[tile_ty] * n_workers,
        names=[f"out{worker}" for worker in range(n_workers)],
    )
)
```

### Worker実装

最初のWorkerの実装では、配列演算が可能なPythonスクリプトを参照するだけです。

**単一Worker:**

```python
# コアが実行するタスク
def core_fn(of_in, of_out):
    elem_in = of_in.acquire(1)
    elem_out = of_out.acquire(1)
    for i in range_(data_size):
        elem_out[i] = elem_in[i] + 1
    of_in.release(1)
    of_out.release(1)


# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, [of_in1.cons(), of_out1.prod()])
```

マルチWorker設計では、同じカーネルが異なるオブジェクトサイズで3回インスタンス化されます。

**複数のWorker:**

```python
# タスクを実行するWorkerを作成
workers = []
for worker in range(n_workers):
    workers.append(
        Worker(
            core_fn,
            [
                of_ins[worker].cons(),
                of_outs[worker].prod(),
            ],
        )
    )
```

### ランタイムシーケンス

最後に、ランタイムシーケンスは、AIE配列との間でデータを移動させるホスト実行コードを定義します。

**単一Workerランタイム:**

```python
# AIE配列との間でデータを移動するランタイム操作
rt = Runtime()
with rt.sequence(data_size, data_size, data_size) as (a_in, b_out, _):
    rt.start(my_worker)
    rt.fill(of_in.prod(), a_in)
    rt.drain(of_out.cons(), b_out, wait=True)
```

マルチWorker設計では、`rt.start()`への呼び出しが少し変わるだけです。

**マルチWorkerランタイム:**

```python
# AIE配列との間でデータを移動するランタイム操作
rt = Runtime()
with rt.sequence(data_size, data_size, data_size) as (a_in, b_out, _):
    rt.start(*workers)
    rt.fill(of_in.prod(), a_in)
    rt.drain(of_out.cons(), b_out, wait=True)
```

### コンパイル

両方の設計を次のコマンドでコンパイルして実行できます：
```bash
make all
```

## 明示的配置レベルIRON

### タイル宣言

最初の設計（[aie2_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2e/aie2_placed.py)）では、3つのタイルを宣言します：1つのコンピュートタイル、1つのメモリタイル、1つのshimタイルです。これらのタイルのリソースは、それぞれデータ処理、ストレージ、外部メモリアクセスの目的で割り当てられます。

**単一コア設計のタイル:**

```python
ShimTile = tile(0, 0)
MemTile = tile(0, 1)
ComputeTile = tile(0, 2)
```

マルチコア設計（[aie2_placed_multi.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2e/aie2_placed_multi.py)）では、1つのコンピュートタイルの代わりに3つのコンピュートタイルを宣言します。

**マルチコア設計のタイル:**

```python
n_cores = 3

ShimTile = tile(0, 0)
MemTile = tile(0, 1)
ComputeTiles = [tile(0, 2 + i) for i in range(n_cores)]
```

### データ移動の設定

次に、タイル間のデータ移動を設定します。高レベルIRONでのアプローチと同様に、最初の設計では4つのObject FIFOを使用し、マルチコア設計ではObject FIFOリンクを使用します。

**単一コア設計のObject FIFO:**

```python
data_size = 48
buffer_depth = 2
data_ty = np.ndarray[(48,), np.dtype[np.int32]]


# 入力データ移動

of_in = object_fifo("in", ShimTile, MemTile, buffer_depth, data_ty)
of_in0 = object_fifo("in0", MemTile, ComputeTile, buffer_depth, data_ty)
object_fifo_link(of_in, of_in0)


# 出力データ移動

of_out = object_fifo("out", MemTile, ShimTile, buffer_depth, data_ty)
of_out0 = object_fifo("out0", ComputeTile, MemTile, buffer_depth, data_ty)
object_fifo_link(of_out0, of_out)
```

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/SingleDesign.svg" height="300" width="700">

マルチコア設計のObject FIFOは、最初の設計から大きく変更されています（[分散と結合パターンを参照](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2b/03_Implicit_Copy/README.md)）。

**マルチコア設計のObject FIFO:**

```python
n_cores = 3
data_size = 48
tile_size = data_size // 3

buffer_depth = 2
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]
tile_ty = np.ndarray[(tile_size,), np.dtype[np.int32]]

# 入力データ移動
inX_fifos = []

of_in = object_fifo("in", ShimTile, MemTile, buffer_depth, data_ty)
for i in range(n_cores):
    inX_fifos.append(object_fifo(
        f"in{i}", MemTile, ComputeTiles[i], buffer_depth, tile_ty
    ))

# 入力/出力データの結合/分散のためのオフセットを計算
if n_cores > 1:
    of_offsets = [16 * i for i in range(n_cores)]
else:
    of_offsets = []
object_fifo_link(of_in, inX_fifos, [], of_offsets)


# 出力データ移動
outX_fifos = []

of_out = object_fifo("out", MemTile, ShimTile, buffer_depth, data_ty)
for i in range(n_cores):
    outX_fifos.append(object_fifo(
        f"out{i}", ComputeTiles[i], MemTile, buffer_depth, tile_ty
    ))
object_fifo_link(outX_fifos, of_out, of_offsets, [])
```

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/MultiDesign.svg" width="1000">

### コア実装

この段階では、各コンピュートタイルがどのコードを実行するかを指定します。

**単一コア:**

```python
@core(ComputeTile)
def core_body():
    # 実質的にwhile(1)
    for _ in range_(0xFFFFFFFF):
        elem_in = of_in0.acquire(ObjectFifoPort.Consume, 1)
        elem_out = of_out0.acquire(ObjectFifoPort.Produce, 1)
        for i in range_(data_size):
            elem_out[i] = elem_in[i] + 1
        of_in0.release(ObjectFifoPort.Consume, 1)
        of_out0.release(ObjectFifoPort.Produce, 1)
```

マルチコア設計では、各コアは前のステップのObject FIFO設定に応じて、適切なObject FIFOのインデックスを使用します。重要な違いは、`data_size`ではなく`tile_size`を処理することです。

**マルチコア:**

```python
for i in range(n_cores):
    # コンピュートタイル i
    @core(ComputeTiles[i])
    def core_body():
        for _ in range_(0xFFFFFFFF):
            elem_in = inX_fifos[i].acquire(ObjectFifoPort.Consume, 1)
            elem_out = outX_fifos[i].acquire(ObjectFifoPort.Produce, 1)
            for j in range_(tile_size):
                elem_out[j] = elem_in[j] + 1
            inX_fifos[i].release(ObjectFifoPort.Consume, 1)
            outX_fifos[i].release(ObjectFifoPort.Produce, 1)
```

### コンパイル

両方の設計を次のコマンドでコンパイルして実行できます：
```bash
make placed
```

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2e)を参照してください。
