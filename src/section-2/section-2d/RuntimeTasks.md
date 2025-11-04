# Section 2d - ランタイムデータ移動（RuntimeTasks）

IRONは、Workerを起動し、外部メモリとの間でObject FIFOにデータを充填および排出する`RuntimeTasks`でプログラムできる`sequence()`関数を持つ`Runtime`クラスを提案します。このセクションで紹介されるすべてのIRON構造は[こちら](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/)で利用できます。

Runtimeシーケンスを作成するために、ユーザーは次のように記述できます：

```python
# AIE配列との間のランタイムデータ移動
rt = Runtime()
with rt.sequence(data_ty_a, data_ty_b, data_ty_c) as (a, b, c):
    # ランタイムタスク
```

この関数の引数は、ホスト側で利用可能なバッファを記述します。関数の本体は、それらのバッファがAIE配列にどのように移動されるかを記述します。

## ランタイムタスク

ランタイムタスクはランタイム中に実行され、同期または非同期の場合があります。タスクは、IRONデザインの作成中にランタイムシーケンスに追加でき、ランタイム中にもキューに入れることができます。

`start()`操作は、IRONデザインで宣言された1つまたは複数のWorkerを開始するために使用されます。以下に示され、[runtime.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/runtime.py)で定義されています：

```python
def start(self, *args: Worker)
```

複数のWorkerが入力として与えられた場合、それらは順番に開始されます。

以下のコードスニペットは、1つの`my_worker` Workerが開始される方法を示しています：

```python
rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):
    rt.start(my_worker)
```

この操作を1回使用して複数のWorkerを開始するには、ユーザーは次のように記述できます：

```python
workers = []
# Workerを作成して"workers"配列に追加

rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):
    rt.start(*workers)
```

`fill()`操作は、`source`ランタイムバッファからのデータでプロデューサ型の`in_fifo` `ObjectFifoHandle`を充填するために使用されます。以下に示され、[runtime.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/runtime.py)で定義されています：

```python
def fill(
        self,
        in_fifo: ObjectFifoHandle,
        source: RuntimeData,
        tap: TensorAccessPattern | None = None,
        task_group: RuntimeTaskGroup | None = None,
        wait: bool = False,
        placement: PlacementTile = AnyShimTile,
    )
```

`wait`入力が`True`に設定されている場合、この操作は待機されます。つまり、操作が終了したときにコントローラが待機しているトークンが生成されます。`placement` Shimタイルも明示的に指定できます。そうでない場合、コンパイラは配置アルゴリズムに基づいて選択します。`task_group`はこのセクションでさらに説明されます。

以下のコードスニペットは、ソースランタイムバッファ`a_in`からのデータが`of_in`のプロデューサ`ObjectFifoHandle`に送信される方法を示しています。このデータは、同じObject FIFOのコンシューマ`ObjectFifoHandle`を介して読み取ることができます。

```python
rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (a_in, _, _):
    rt.fill(of_in.prod(), a_in)
```

`drain()`操作は、データのコンシューマ型の`out_fifo` `ObjectFifoHandle`を充填し、そのデータを`dest`ランタイムバッファに書き込むために使用されます。以下に示され、[runtime.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/runtime.py)で定義されています：

```python
def drain(
    self,
    out_fifo: ObjectFifoHandle,
    dest: RuntimeData,
    tap: TensorAccessPattern | None = None,
    task_group: RuntimeTaskGroup | None = None,
    wait: bool = False,
    placement: PlacementTile = AnyShimTile,
)
```

`wait`入力が`True`に設定されている場合、この操作は待機されます。つまり、操作が終了したときにコントローラが待機しているトークンが生成されます。`placement` Shimタイルも明示的に指定できます。そうでない場合、コンパイラは配置アルゴリズムに基づいて選択します。`task_group`はこのセクションでさらに説明されます。

以下のコードスニペットは、`of_out`のコンシューマ`ObjectFifoHandle`からのデータが宛先ランタイムバッファ`c_out`に排出される方法を示しています。データは、そのプロデューサ`ObjectFifoHandle`を介して`of_out`に生成できます。さらに、`drain()`タスクの`wait`入力が設定されているため、このタスクは完了まで待機されます。つまり、`c_out`ランタイムバッファが`data_ty`で記述されているように十分なデータを受信するまでです。

```python
rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, c_out):
    rt.drain(of_out.cons(), c_out, wait=True)
```

## ランタイムシーケンスへの操作のインライン化

場合によっては、任意のMLIR操作を生成するPython関数をランタイムシーケンスに挿入することが望ましい場合があります。そのような例の1つは、ユーザーがランタイムパラメータを設定したい場合です。これらは、ランタイム時にWorkerのローカルメモリモジュールにロードされます。

ランタイムシーケンスに操作をインライン化するには、ユーザーは`inline_ops()`操作を使用できます。以下に示され、[runtime.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/runtime.py)で定義されています：

```python
def inline_ops(self, inline_func: Callable, inline_args: list)
```

`inline_func`はMLIRコンテキスト内で実行する関数であり、`inline_args`は関数が実行するために必要な状態です。

次のコードスニペットでは、`GlobalBuffers`の配列が作成され、各バッファは`16xi32`型のランタイムパラメータを保持します。[`GlobalBuffer`](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/globalbuffer.py)は、IRONデザインのトップレベルで宣言されたメモリ領域で、Workerとランタイムの両方が操作に利用できます。`use_write_rtp`が設定されている場合、ランタイムパラメータ固有の操作が、コンパイラ抽象化の下位レベルでランタイムシーケンス内に生成されます。

```python
# ランタイムパラメータ
rtps = []
for i in range(4):
    rtps.append(
        GlobalBuffer(
            np.ndarray[(16,), np.dtype[np.int32]],
            name=f"rtp{i}",
            use_write_rtp=True,
        )
    )
```

ランタイムパラメータの実際の値は、ランタイム時に各バッファにロードされます：

```python
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

これらのグローバルバッファへのデータの伝播は瞬時ではなく、Workerがランタイムパラメータが利用可能になる前にそれらを読み取る可能性があります。これを解決するために、[worker.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/worker.py)で定義されている`WorkerRuntimeBarrier`をインスタンス化することができます：

```python
class WorkerRuntimeBarrier:
    def __init__(self, initial_value: int = 0)
```

これらのバリアは、個々のWorkerがランタイムシーケンスと同期できるようにします：

```python
workerBarriers = []
for i in range(4):
    workerBarriers.append(WorkerRuntimeBarrier())

...

def core_fn(of_in, of_out, rtp, barrier):
    barrier.wait_for_value(1)
    runtime_parameter = rtp

...

rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):

    ...

    rt.inline_ops(set_rtps, rtps)

    for i in range(4):
        rt.set_barrier(workerBarriers[i], 1)
```

現在、`WorkerRuntimeBarrier`は0から63の間の任意の値を取ることができます。これは、これらのバリアがアーキテクチャのロックメカニズムを内部で活用しているためです。

> **注意：** `GlobalBuffer`と同様に、単一のバリアを作成して複数のWorkerへの入力として渡すことができます。コンパイラ抽象化の下位ステージでは、これにより各Workerに対して異なるロックが使用されます。

## ランタイムタスクグループ

ランタイムシーケンスを再構成し、以前の構成からいくつかのリソースを再利用することが望ましい場合があります。特に、DMAタスクキュー内のBDなどの一部のリソースが制限されている場合です。

この再構成ステップを容易にするために、IRONは`RuntimeTaskGroup`を導入します。これは、[runtime.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/iron/runtime/runtime.py)で定義されている`task_group()`関数を使用して作成できます。

`RuntimeTask`は、`task_group`入力を指定することでタスクグループに追加できます。同じグループ内のタスクは、ランタイムシーケンスに追加され、順番に実行されます。`finish_task_group()`操作は、タスクグループの終わりをマークするために使用されます。つまり、この操作の後、グループ内のすべてのタスクが完了まで待機され、その後すべてが同時に解放され、ランタイムシーケンスが次のタスクグループによって再構成されます。

> **注意：** 完了までランタイムタスクを待機し、すべてのリソースを同時に解放する能力により、タスクグループはランタイムデータ移動タスクの非同期性を処理するのに適しています。

以下のコードスニペットのランタイムシーケンスには、2つのタスクグループがあります。2番目のタスクグループの作成は、最初のタスクグループの実行の終わりに発生することがわかります。

```python
rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (a_in, _, c_out):
    rt.start(*workers)

    tg = rt.task_group() # 最初のタスクグループを開始
    for groups in [0, 1]:
        rt.fill(of_in.prod(), a_in, task_group=tg)
        rt.drain(of_out.cons(), c_out, task_group=tg, wait=True)
        rt.finish_task_group(tg)
        tg = rt.task_group() # 2番目のタスクグループを開始
    rt.finish_task_group(tg)
```

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2d)を参照してください。
