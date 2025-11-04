# IRONミニチュートリアル

## 主要コンポーネント：Workers、ObjectFifos、Runtime

IRONは、NPUプログラミングのための非配置（遅延配置）[API](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/)を提供します。以下は、AIE計算コードとObject FIFOデータ移動プリミティブを説明する例です：

Workerを使用した計算コード：

```python
# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

def core_fn():
    buff = LocalBuffer(
        data_ty,
        name="buff",
        initial_value=np.array(range(data_size), dtype=np.int32),
    )

    for i in range_(data_size):
        buff[i] = buff[i] + 1

# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, []) # 配置を強制できます: placement=Tile(1, 3)
```

Workerの詳細については、プログラミングガイドの[セクション1](../section-1/index.html)および[worker.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/worker.py)を参照してください。

Object FIFOデータ移動プリミティブ：

```python
# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# ObjectFifosを使用したデータフロー
of_in = ObjectFifo(data_ty, name="in") # デフォルトの深さは2
```

Object FIFOの詳細については、プログラミングガイドの[セクション2a](../section-2/section-2a/index.html)および[objectfifo.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/dataflow/objectfifo.py)を参照してください。

このミニチュートリアルのIRONコード[例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/aie2p.py)は、IRONデザインのさまざまな部分を詳述しています。特にRuntimeの詳細については、プログラミングガイドの[セクション2d](../section-2/section-2d/index.html)を参照してください。

## 演習

1. [exercise_1](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_1/exercise_1.py)に慣れてください。コードには、既にインスタンス化されたローカルバッファを持つ単一のWorkerが含まれており、それを外部メモリに送信します。`python3 exercise_1.py`を実行してプログラムを実行し、出力を確認してください。

2. [exercise_1](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_1/exercise_1.py)のコードを修正して、Workerがローカルバッファを使用する代わりに外部メモリから入力データを受け取るようにします。つまり、パススルーです。

3. [exercise_1](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_1/exercise_1.py)のコードを修正して、ObjectFifosのみを使用し、Workerを使用せずに、外部メモリからMemタイルを経由して戻るようにデータをルーティングします。このためには、プログラミングガイドの[セクション2b - 暗黙的コピー](../section-2/section-2b/03_Implicit_Copy/index.html)で説明されている`forward()`関数が必要です。[セクション2d](../section-2/section-2d/RuntimeTasks.html)で説明されているように、ランタイムで入力Object FIFOにデータを`fill()`することを忘れないでください。

4. [exercise_1](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_1/exercise_1.py)のコードを修正して、最初に外部メモリからMemタイルを経由してWorkerにデータをルーティングし、Workerがパススルーを実行してから、データを送り返すようにします。

5. [exercise_1](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_1/exercise_1.py)のコードを修正して、WorkerのoutputがMemタイルを経由してルーティングされるようにします。

## 複雑なデータ移動パターン：Broadcast、Split、Join

IRONデザインは、複数のWorkerを使用するように簡単にスケーリングできます：

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

def core_fn(...):
    # ...カーネルコード...

# タスクを実行するWorkerを作成
workers = []
for _ in range(n_workers):
    workers.append(
        Worker(core_fn, [...])
    )

rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):
    rt.start(*workers)
```

複数のWorkerのプログラミングの詳細については、プログラミングガイドの[セクション2e](../section-2/section-2e/index.html)を参照してください。

broadcast、split、joinなどの複雑なデータ移動パターンは、`ObjectFifo`、特にプロデューサまたはコンシューマハンドルのいずれかである`ObjectFifoHandle`を使用してサポートされます。これらは、複数のコンシューマを持つブロードキャストパターンを決定するために使用されます。

Broadcast - ブロードキャストに関する詳細なドキュメントは、プログラミングガイドの[セクション2b - Broadcast](../section-2/section-2b/02_Broadcast/index.html)で利用できます。

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# ObjectFifosを使用したデータフロー
of_in = ObjectFifo(data_ty, name="in")

def core_fn(of_in):
    # ...カーネルコード...

# タスクを実行するWorkerを作成
workers = []
for _ in range(n_workers):
    workers.append(
        Worker(core_fn, [of_in.cons()]) # of_in.cons()を呼び出すたびに新しいObjectFifoHandleが返されます
    )
```

`split()`および`join()`メソッドは、それぞれ複数の出力および入力`ObjectFifos`を作成するために使用されます。

Split - `split()`に関する詳細なドキュメントは、プログラミングガイドの[セクション2b - 暗黙的コピー](../section-2/section-2b/03_Implicit_Copy/index.html)で利用できます。

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]
tile_size = data_size // n_workers
tile_ty = np.ndarray[(tile_size,), np.dtype[np.int32]]

# ObjectFifosを使用したデータフロー
of_offsets = [tile_size * worker for worker in range(n_workers)]

of_in = ObjectFifo(data_ty, name="in")
of_ins = of_in.cons().split(
    of_offsets,
    obj_types=[tile_ty] * n_workers,
    names=[f"in{worker}" for worker in range(n_workers)],
)

def core_fn(of_in):
    # ...カーネルコード...

# タスクを実行するWorkerを作成
workers = []
for worker in range(n_workers):
    workers.append(
        Worker(core_fn, [of_ins[worker].cons()])
    )
```

Join - `join()`に関する詳細なドキュメントは、プログラミングガイドの[セクション2b - 暗黙的コピー](../section-2/section-2b/03_Implicit_Copy/index.html)で利用できます。

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]
tile_size = data_size // n_workers
tile_ty = np.ndarray[(tile_size,), np.dtype[np.int32]]

# ObjectFifosを使用したデータフロー
of_offsets = [tile_size * worker for worker in range(n_workers)]

of_out = ObjectFifo(data_ty, name="out")
of_outs = of_out.prod().join(
    of_offsets,
    obj_types=[tile_ty] * n_workers,
    names=[f"out{worker}" for worker in range(n_workers)],
)

def core_fn(of_out):
    # ...カーネルコード...

# タスクを実行するWorkerを作成
workers = []
for worker in range(n_workers):
    workers.append(
        Worker(core_fn, [of_outs[worker].prod()])
    )
```

Object FIFOデータ移動パターンの詳細については、プログラミングガイドの[セクション2b](../section-2/section-2b/index.html)を参照してください。

## 演習

1. [exercise_2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_2/exercise_2.py)に慣れてください。[exercise_2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_2/exercise_2.py)のコードを修正して、入力データが3つのWorker間で分割され、それらの出力が結合されてから最終結果が外部メモリに送信されるようにします。

2. [exercise_2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_2/exercise_2.py)のコードを修正して、各Workerのデータサイズが不均等になるようにします。たとえば：tile_sizes = [8, 24, 16]。

## ランタイムシーケンス

IRONランタイムシーケンスの引数は、ホスト側で利用可能なバッファを記述します。シーケンスの本体には、`ObjectFifos`を通じてそれらのバッファがAIE配列にどのように移動されるかを記述するコマンドが含まれています。

```python
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# ObjectFifosを使用したデータフロー
of_in = ObjectFifo(data_ty, name="in")
of_out = ObjectFifo(data_ty, name="out")

rt = Runtime()
with rt.sequence(tile_ty, tile_ty) as (a_in, c_out):
    rt.start(my_worker)
    rt.fill(of_in.prod(), a_in)
    rt.drain(of_out.cons(), c_out, wait=True)
```

ランタイムシーケンスでは最大5つのバッファがサポートされており、5番目は通常トレースサポートに使用されます。これについては、プログラミングガイドの[セクション4b](../section-4/section-4b/index.html)でさらに説明されています。

ランタイムシーケンスコマンドは、専用のコマンドプロセッサに順番に送信され、実行されます。コマンドプロセッサは、完了に関連付けられたトークンが生成されるまで、`wait`に設定されているコマンドを待機します。ランタイムシーケンス内のすべてのコマンドが実行されると、コマンドプロセッサはホストプロセッサに割り込みを送信します。

IRONは、`task_group`を使用したランタイムシーケンスコマンドのグループ化もサポートしています。同じグループ内のコマンドは同時に実行を開始し、グループの完了は`finish_task_group()`コマンドを使用して明示的に同期できます。これらの機能を組み合わせて、並列タスクの待機の最適化されたグループ化を実現できます。これは[この](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/memcpy/)プログラミング例に示されています。

ランタイムシーケンスの詳細については、プログラミングガイドの[セクション2d](../section-2/section-2d/RuntimeTasks.html)を参照してください。

## 演習

1. [exercise_3](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_3/exercise_3.py)に慣れてください。現在、デザインは単純なパススルー、つまり`out_C = in_A`を行っており、ランタイムシーケンス内の`drain()`コマンドによって完了時にトークンが発行されます。`fill()`と`drain()`コマンドの場所を入れ替えて、`python3 exercise_3.py`を実行してください。何が起こるか観察してください。

2. [exercise_3](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_3/exercise_3.py)のコードを元のバージョンに復元してください。[exercise_3](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_3/exercise_3.py)のコードを修正して、外部メモリからの2つの入力テンソルの加算を行うようにします。つまり`out_C = in_A + in_B`。

## ランタイムパラメータとバリア

IRONは、ランタイム時に設定されWorkerに伝播されるランタイムパラメータをサポートしています。

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# ランタイムパラメータ
rtps = []
for i in range(n_workers):
    rtps.append(
        GlobalBuffer(
            np.ndarray[(16,), np.dtype[np.int32]],
            name=f"rtp{i}",
            use_write_rtp=True,
        )
    )

def core_fn(rtp):
    runtime_parameter = rtp

# タスクを実行するWorkerを作成
workers = []
for worker in range(n_workers):
    workers.append(
        Worker(core_fn, [rtps[worker]])
    )

rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):

    # ランタイムパラメータを設定
    def set_rtps(*args):
        for rtp in args:
            rtp[0] = 50
            rtp[1] = 255
            rtp[2] = 0

    rt.inline_ops(set_rtps, rtps)
```

RTPが早期に読み取られないようにするために、`WorkerRuntimeBarriers`を使用してWorkerをランタイムシーケンスと同期できます：

```python
n_workers = 4

# テンソル型を定義
data_size = 256
data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

# ランタイムパラメータ
# ...

# Workerランタイムバリア
workerBarriers = []
for _ in range(n_workers):
    workerBarriers.append(WorkerRuntimeBarrier())

def core_fn(rtp, barrier):
    barrier.wait_for_value(1)
    runtime_parameter = rtp

# タスクを実行するWorkerを作成
workers = []
for worker in range(n_workers):
    workers.append(
        Worker(core_fn, [rtps[worker], workerBarriers[worker]])
    )

rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):
    # ランタイムパラメータを設定
    # ...
    rt.inline_ops(set_rtps, rtps)

    for worker in range(n_workers):
        rt.set_barrier(workerBarriers[worker], 1)
```

ランタイムパラメータとバリアの詳細については、プログラミングガイドの[セクション2d](../section-2/section-2d/RuntimeTasks.html)および[worker.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/worker.py)を参照してください。

## 演習

1. [exercise_4](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_4/exercise_4.py)に慣れてください。83行目を修正して、`USE_INPUT_VEC = False`に設定してください。`python3 exercise_4.py`を実行してください。

2. Workerがランタイムが設定する前にRTPを読み取るため、デザインは失敗します。[exercise_4](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_4/exercise_4.py)のコードを修正して、WorkerがRTPが設定されるのを待つために`WorkerRuntimeBarrier`を使用するようにします。

## 高度なトピック：データレイアウト変換

AIE配列のDMAは、オンザフライでデータ変換を実行できます。各次元の変換は、(size, stride)のペアとして表現されます。次元は最高から最低まで与えられます：

```python
[(size_2, stride_2), (size_1, stride_1), (size_0, stride_0)]
```

データレイアウト変換は、ハードウェアにデータのどの場所に次にアクセスするかを指定する方法として見ることができ、そのため、一連のネストされたループを使用してアクセスパターンをモデル化することができます。たとえば、上記のstridesとsizesを使用した変換は、次のように表現できます：

```c
int *buffer;
for(int i = 0; i < size_2; i++)
    for(int j = 0; j < size_1; j++)
        for(int k = 0; k < size_0; k++)
            // buffer[  i * stride_2
            //        + j * stride_1
            //        + k * stride_0]の要素にアクセス/格納
```

**ランタイム時**のDMAオンザフライデータ変換をより適切にサポートするために、IRONは`Tensor Access Pattern`（`taps`）の構成要素を提供する[taplib](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/helpers/taplib/)を提供します。sizesとstridesはグループ化され、次元は最高から最低まで与えられる必要があります（最大4次元）：

```python
tap = TensorAccessPattern(
    tensor_dims=(2, 3),
    offset=0,
    sizes=[size_1, size_0],
    strides=[stride_1, stride_0],
)
```

`taps`にはさらにオフセットがあるため、`tensor_dims`は元のテンソルのサイズより小さくなる場合があります。

`TensorAccessPattern`は、`fill()`および`drain()`ランタイム操作に適用できます：

```python
rt = Runtime()
with rt.sequence(data_ty, data_ty) as (a_in, c_out):
    rt.start(my_worker)
    rt.fill(of_in.prod(), a_in, tap)
    rt.drain(of_out.cons(), c_out, tap, wait=True)
```

`TensorAccessPattern`は、2つの方法で可視化できます：
- 要素がアクセスされる順序を示すヒートマップ
- `TensorAccessPattern`によってテンソル内の各要素がアクセスされる回数を示すヒートマップ

```python
tap.visualize(show_arrows=True, plot_access_count=True)
```

`taps`は`TensorAccessSequence`にグループ化でき、各`tap`は（タイリングパターンからの）異なるタイルを表します：

```python
t0 = TensorAccessPattern((8, 8), offset=0, sizes=[1, 1, 4, 4], strides=[0, 0, 8, 1])
t1 = TensorAccessPattern((8, 8), offset=4, sizes=[1, 1, 4, 4], strides=[0, 0, 8, 1])
t2 = TensorAccessPattern((8, 8), offset=32, sizes=[1, 1, 4, 4], strides=[0, 0, 8, 1])

taps = TensorAccessSequence.from_taps([t0, t1, t2])
```

次に、`taps`はシーケンス内で配列としてアクセスできます：

```python
for t in taps:
```

`tap`のsizesとstridesを推測することは、ユーザーにとって困難な場合があります。`taplib`は、この課題に対処しようとする`TensorTiler2D`クラスを導入します。Tilerは、一般的なタイリングパターンの`taps`を生成するように設計された探索的機能です。Tilerは、生成された`taps`を`TensorAccessSequence`として返します：

```python
tensor_dims = (8, 8)
tile_dims = (4, 4)
simple_tiler = TensorTiler2D.simple_tiler(tensor_dims, tile_dims)
```

上記のsimple tilerは、タイリングに対して非常に簡単なアプローチを取り、与えられた次元に基づいてデータの垂直分割を行います。より多くのtilerは[tensortiler2d.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/helpers/taplib/tensortiler2d.py)で利用できます。

`taplib`の詳細については、[tiling_exploration](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/tiling_exploration/)を参照してください。

`ObjectFifo`は、`dims_to_stream`および`dims_from_stream_per_cons`入力を介してDMAオンザフライデータ変換を表現できます。これらの入力は、ペアのリストとして構造化されており、各ペアはDMA変換の次元に対して(size, stride)として表現されます。次元は最高から最低まで与えられる必要があります：

```python
dims = [(size_2, stride_2), (size_1, stride_1), (size_0, stride_0)]
of_out = ObjectFifo(data_ty, name="out", dims_to_stream=dims)
```

オフセットは現在Object FIFOレベルでは表現されていないため、次元はオブジェクトのフルサイズに適用可能である必要があります。

Object FIFOデータレイアウト変換の詳細については、プログラミングガイドの[セクション2c](../section-2/section-2c/index.html)を参照してください。

## 演習

1. [exercise_5a](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5a/exercise_5a.py)に慣れてください。ランタイム時に入力データに対して実行されるデータ変換が[ref_plot.png](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5a/ref_plot.png)に示されているものと一致するように`tap`を使用してください。ランタイム`fill()`操作に`tap`を追加することを忘れないでください。例を実行する前に、83行目を`USE_REF_VEC = False`に修正してください。`python3 exercise_5a.py`を実行して答えを確認してください。

2. [exercise_5a](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5a/exercise_5a.py)で追加した`tap`を、`TensorTiler2D`によって生成されたものに置き換えてください。これには、[tensortiler2d.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/helpers/taplib/tensortiler2d.py)で定義されている`simple_tiler()`コンストラクタが必要です。`python3 exercise_5a.py`を実行して答えを確認してください。2つの生成されたプロットも観察できます。

3. [exercise_5a](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5a/exercise_5a.py)のコードを修正して、ランタイム時ではなく、`of_out`に直接データ変換が適用されるようにします。`python3 exercise_5a.py`を実行して答えを確認してください。

4. [exercise_5b](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5b/exercise_5b.py)に慣れてください。`TensorAccessSequence`内の`taps`が[exercise_5a](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial/exercise_5/exercise_5a/exercise_5a.py)のものとわずかに異なることを観察してください。`python3 exercise_5b.py`を実行し、2つの生成されたプロットを観察してください。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/mini_tutorial)を参照してください。
