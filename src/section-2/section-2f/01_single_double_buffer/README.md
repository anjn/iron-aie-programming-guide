# シングル/ダブルバッファ

[single_buffer.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/01_single_double_buffer/single_buffer.py)のデザインは、Object FIFO `of_in`を使用して`my_worker`の出力を`my_worker2`に転送し、Object FIFO `of_out`を使用して`my_worker2`の出力を外部メモリに転送します。`of_in`の深さは`1`で、以下の図に示すように、2つのWorker間の単一バッファを記述しています。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/SingleBuffer.svg" height=200 width="500">

> **注意：** 上の図は、Workerが既に`ComputeTile2`と`ComputeTile3`にマッピングされていることを前提としています。ただし、これが唯一可能なマッピングではなく、Workerを作成する際、その配置はコンパイラに任せることができます。

このデザインのプロデューサおよびコンシューマプロセスの両方が、些細なタスクを持っています。`my_worker`で実行されているプロデューサプロセスは、単一バッファをacquireし、消費のためにreleaseする前にそのすべてのエントリに`1`を書き込みます。`my_worker2`で実行されているコンシューマプロセスは、`of_in`から単一バッファと`of_out`から単一バッファをacquireし、入力Object FIFOから出力Object FIFOにデータをコピーし、他のプロセスのために両方のオブジェクトをreleaseします。

このデザインでデータ転送にダブルバッファ（またはピンポンバッファ）を使用するには、ユーザーはObject FIFOの深さを`2`に設定するだけです。Object FIFOの低レベル化がピンバッファとポンバッファの間を適切に循環させるため、他の変更は必要ありません。深さを変更するには、ユーザーは次のように記述する必要があります：

```python
of_in = ObjectFifo(data_ty, name="in", depth=2) # ダブルバッファ
of_out = ObjectFifo(data_ty, name="out", depth=2) # ダブルバッファ
```

この変更により、以下の図に示すように、Object FIFOの利用可能なリソースの数が効果的に増加します：

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/DoubleBuffer.svg" height=200 width="500">

[programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)で利用可能なすべての例には、このデータ移動パターンが含まれています。

このデザインをコンパイル、実行、テストするには、次のコマンドを使用できます：

```bash
make
make run
```

このデザインの明示的に配置されたレベルのIRONプログラミングは、[single_buffer_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/01_single_double_buffer/single_buffer_placed.py)で利用できます。次のコマンドでコンパイル、実行、テストできます：

```bash
env use_placed=1 make
make run
```

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/01_single_double_buffer)を参照してください。
