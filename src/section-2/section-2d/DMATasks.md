# Section 2d - ランタイムデータ移動（DMATasks）

## `npu_dma_memcpy_nd`による効率的なデータ移動

`npu_dma_memcpy_nd`関数は、AI Engine配列と外部メモリ間の異なるメモリ領域間で、ノンブロッキングの多次元データ転送を可能にするための鍵となります。この関数は、信号処理、機械学習、ビデオ処理などの実際のアプリケーションを開発する上で不可欠です。

**関数シグネチャとパラメータ**：

```python
npu_dma_memcpy_nd(metadata, bd_id, mem, offsets=None, sizes=None, strides=None)
```

- **`metadata`**: これは、ホスト側のメモリ転送に割り当てられたShim Tileとそのその1つのDMAチャネルを記録するObject FIFOへの参照、またはObject FIFOの文字列名です。memcpy操作をObject FIFOに関連付けるために、このメタデータ文字列はObject FIFO名文字列と一致する必要があります。
- **`bd_id`**: このmemcpyに使用される特定のバッファディスクリプタ制御レジスタの識別整数。バッファディスクリプタには、以下のパラメータで説明されるDMA転送に必要なすべての情報が含まれています。
- **`mem`**: シーケンス関数への引数として与えられたホストバッファへの参照。この転送はこのバッファから読み取るか書き込みます。
- **`tap`**（オプション）: `TensorAccessPattern`は、`mem`バッファ上のアクセスパターンを決定するための`offset`/`sizes`/`strides`を指定する代替方法です。
- **`offsets`**（オプション）: 各次元でのデータ転送の開始点。最大4つのオフセット次元があります。
- **`sizes`**: 各次元で転送されるデータの範囲。最大4つのサイズ次元があります。
- **`strides`**（オプション）: 各次元のデータポイント間の間隔ステップ。ストライドとデータの再形成に役立ちます。
- **`burst_length`**（オプション）: DMAタスクのバースト長の構成。`0`の場合、利用可能な最高値にデフォルト設定されます。

stridesとsizesは、[セクション2C](../section-2c/index.html)で説明されているものと同様にデータ変換を表現します。

**使用例**：

```python
npu_dma_memcpy_nd(of_in, 0, input_buffer, sizes=[1, 1, 1, 30])
```

上記の例は、ホストメモリの`input_buffer`から30データ要素（120バイト）の線形転送を、「of_in」とラベル付けされた一致するメタデータを持つObject FIFOに記述しています。`size`次元は右から左に表現され、右が次元0で左が次元3です。使用されない上位次元は`1`に設定する必要があります。

## 多次元`npu_dma_memcpy_nd`の高度な技術

AMDのAI Engineでの高性能コンピューティングアプリケーションでは、複雑なデータ移動のための`npu_dma_memcpy_nd`関数を習得することが重要です。ここでは、`sizes`、`strides`、`offsets`パラメータを使用して、複雑なデータ転送を効果的に管理することに焦点を当てます。

### 大きな行列のタイル化

2D行列のタイル化などの一般的なタスクは、`npu_dma_memcpy_nd`操作を使用して実装できます。ここに説明を示す簡単な例があります。

**シナリオ**: 2D行列を[100, 200]から[20, 20]の形状にタイル化し、データ型は`int16`です。規約は[row, col]です。

**1. 1つのタイルを転送する構成**：

```python
metadata = of_in
bd_id = 3
mem = matrix_memory  # 行列のメモリオブジェクト

# サイズはコピーするタイルの範囲を定義します
sizes = [1, 1, 20, 10]

# ストライドは上位（未使用）次元では'0'に、下位次元では'100'（4Bまたは"i32s"での行の長さ）に設定されます
strides = [0, 0, 0, 100]

# オフセットは先頭から開始するためゼロに設定されます
offsets = [0, 0, 0, 0]

npu_dma_memcpy_nd(metadata, bd_id, mem, offsets, sizes, strides)
```

**2. 行列全体をタイル化する構成**：

```python
metadata = of_in
bd_id = 3
mem = matrix_memory  # 行列のメモリオブジェクト

# サイズはコピーするタイルの範囲を定義します。
# 次元0は10で、タイルの1行に対して20個のint16を転送します、
# 次元1はその行転送を20回繰り返して[20, 20]タイルを完成させます、
# 次元2はそのタイル転送を行に沿って10回繰り返します、
# 次元3はタイルの行の転送を5回繰り返して完成させます。
sizes = [5, 10, 20, 10]

# ストライドは最高（未使用）次元では'0'に、
# 次の行のタイルには'2000'（200 x 20 x 2B / 4B）に、
# 最後の[20, 20]タイルの「右」の次のタイルには'10'に、
# 次元0では'100'（4Bまたは"i32s"での行の長さ）に設定されます。
strides = [0, 2000, 10, 100]

# オフセットは先頭から開始するためゼロに設定されます
offsets = [0, 0, 0, 0]

npu_dma_memcpy_nd(metadata, bd_id, mem, offsets, sizes, strides)
```

## 1つ以上の`npu_dma_memcpy_nd`操作後の`dma_wait`によるホスト同期

DMAチャネルとホスト間の同期は、`dma_wait`操作によって促進され、データの整合性と適切な実行順序が保証されます。`dma_wait`操作は、ObjectFifoに関連付けられたBDが完了するまで待機し、タスク完了トークンを発行します。

**関数シグネチャ**：

```python
dma_wait(metadata)
```

- **`metadata`**: 待機するDMAオプションに関連付けられたObjectFifo pythonオブジェクトまたはObject FIFOの名前。

**使用例**：

1つのObject FIFOに関連付けられたDMAを待機：

```python
# 出力データが出力Object FIFOからホストに転送されるのを待ちます
dma_wait(of_out)
```

複数のObject FIFOに関連付けられたDMAを待機：

```python
dma_wait(of_in, of_out)
```

## `npu_dma_memcpy_nd`によるデータ移動と同期のベストプラクティス

- **バッファディスクリプタを再利用するための同期**: 各`npu_dma_memcpy_nd`には`bd_id`が割り当てられます。各Shim Tileで使用可能なBDは最大`16`個です。すべての転送が完了したらBDを再利用することが「安全」です。これは、計算操作を完了するために配列にデータを転送するために完了する必要があるBDを考慮して、適切に同期することで管理できます。次に、計算操作によって生成されたデータを受信するBDで同期して、ホストメモリに書き戻します。
- **ノンブロッキング転送の注意**: `npu_dma_memcpy_nd`のノンブロッキング性質を活用して、データ転送と計算を重複させます。
- **同期オーバーヘッドを最小化**: 性能を低下させる可能性のある過度のオーバーヘッドを避けるために、慎重に同期/待機します。

## `dma_task`操作による効率的なデータ移動

`npu_dma_memcpy_nd`と`dma_wait`の代替として、同様の目的を果たすことができる**DMAタスク**に関する一連の操作があります。

`npu_dma_memcpy_nd`を使用するよりもDMAタスク操作を使用する利点が2つあります：
* ユーザーはBD番号を指定する必要がありません
* DMAタスク操作はBD操作の*チェーン*が可能です。ただし、これはこのガイドの範囲を超えた高度なユースケースです。

すべてのプログラミング例には、DMAタスク操作を使用して記述された`*_placed.py`バージョンがあります。

**関数シグネチャとパラメータ**：

```python
def shim_dma_single_bd_task(
    alloc,
    mem,
    tap: TensorAccessPatter | None = None,
    offset: int | None = None,
    sizes: MixedValues | None = None,
    strides: MixedValues | None = None,
    transfer_len: int | None = None,
    issue_token: bool = False,
)
```

- **`alloc`**: `alloc`引数はDMAタスクをObjectFIFOに関連付けます。この引数は`alloc`と呼ばれます。これは、データ転送のshim側のエンドポイント（具体的にはshimタイルのチャネル）が、いわゆる「shim DMA allocation」を通じて参照されるためです。ObjectFIFOがShim Tileエンドポイントで作成されると、ObjectFIFOと同じ名前の割り当てが自動的に生成されます。
- **`mem`**: シーケンス関数への引数として与えられたホストバッファへの参照。この転送はこのバッファから読み取るか書き込みます。
- **`tap`**（オプション）: `TensorAccessPattern`は、`mem`バッファ上のアクセスパターンを決定するための`offset`/`sizes`/`strides`を指定する代替方法です。
- **`offset`**（オプション）: データ転送の開始点。デフォルト値は`0`です。
- **`sizes`**: 各次元で転送されるデータの範囲。最大4つのサイズ次元があります。
- **`strides`**（オプション）: 各次元のデータポイント間の間隔ステップ。ストライドとデータの再形成に役立ちます。
- **`issue_token`**（オプション）: トークンが発行された場合、返されたタスクに対して`dma_await_task`を呼び出すことができます。デフォルトは`False`です。
- **`burst_length`**（オプション）: DMAタスクのバースト長の構成。`0`の場合、利用可能な最高値にデフォルト設定されます。

stridesとstridesは、[セクション2C](../section-2c/index.html)で説明されているものと同様にデータ変換を表現します。

**使用例**：

```python
out_task = shim_dma_single_bd_task(of_out, C, sizes=[1, 1, 1, N], issue_token=True)
```

上記の例は、ホストメモリの`C`バッファから「of_out」とラベル付けされた一致するメタデータを持つObject FIFOへの`N`データ要素の線形転送を記述しています。`sizes`次元は右から左に表現され、右が次元0で左が次元3です。使用されない上位次元は`1`に設定する必要があります。

## `dma_await_task`によるホスト同期

DMAチャネルとホスト間の同期は、`dma_await_task`操作によって促進され、データの整合性と適切な実行順序が保証されます。`dma_await_task`操作は、タスクに関連付けられたすべてのBDが完了するまで待機します。

**関数シグネチャ**：

```python
def dma_await_task(*args: DMAConfigureTaskForOp)
```

- `args`: 1つ以上の`dma_task`オブジェクト。`dma_task`オブジェクトは`shim_dma_single_bd_task`によって返される値です。

**使用例**：

1つのDMAタスクのタスク完了を待機：

```python
# 出力タスクが完了するのを待ちます
dma_await_task(out_task)
```

複数のDMAタスクのタスク完了を待機：

```python
# 入力タスクが完了し、次に出力タスクが完了するのを待ちます
dma_await_task(in_task, out_task)
```

## `dma_free_task`で待機せずにBDを解放

`dma_await_task`は、`issue_token=True`で作成されたタスクに対してのみ呼び出すことができます。`issue_token=False`（デフォルト）の場合、プログラマがタスクが完了したことを知っているときに`dma_free_task`を呼び出す必要があります。`dma_free_task`を使用すると、コンパイラは同期なしでタスクのBDを再利用できます。タスク`X`が完了する前に`dma_free_task(X)`を使用すると、競合状態と予測不可能な動作につながります。`dma_free_task(X)`は、他の同期手段と組み合わせてのみ使用してください。たとえば、タスク`Y`がタスク`X`が完了した後にのみ完了できると推論できる場合、`dma_await_task(Y)`の呼び出しの後に`dma_free_task(X)`を発行できます。

**関数シグネチャ**：

```python
def dma_free_task(*args: DMAConfigureTaskForOp)
```

- `args`: 1つ以上の`dma_task`オブジェクト。`dma_task`オブジェクトは`shim_dma_single_bd_task`によって返される値です。

**使用例**：

1つのタスクに関連付けられたDMAに属するBDを解放：

```python
# コンパイラがタスクのBDを再利用できるようにします。プログラマがタスクが完了したことを確認している場合にのみ呼び出す必要があります。
dma_free_task(out_task)
```

複数のタスクに関連付けられたDMAに属するBDを解放：

```python
# コンパイラが複数のタスクのBDを再利用できるようにします。プログラマがすべてのタスクが完了したことを確認している場合にのみ呼び出す必要があります。
dma_free_task(in_task, out_task)
```

## `dma_task`操作によるデータ移動と同期のベストプラクティス

- **バッファディスクリプタを再利用するための待機または解放**: `dma_task`操作では、各操作に使用される正確なバッファディスクリプタ（BD）はユーザーには見えませんが、依然として有限数（Shim Tileで最大`16`）があります。したがって、BDの数が使い果たされる前に`dma_await_task`または`dma_free_task`を使用して再利用できるようにすることが重要です。
- **ノンブロッキング転送の注意**: `dma_start_task`のノンブロッキング性質を活用して、データ転送と計算を重複させます。
- **同期オーバーヘッドを最小化**: 性能を低下させる可能性のある過度のオーバーヘッドを避けるために、慎重に同期/待機します。

## 結論

`npu_dma_memcpy_nd`と`dma_wait`関数は、Ryzen™ AI NPUのAI Engineでデータ転送と同期を管理するための強力なツールです。これらの関数を活用するアプリケーションを理解し効果的に実装することで、開発者は高性能コンピューティングアプリケーションの性能、効率、精度を向上させることができます。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2d)を参照してください。
