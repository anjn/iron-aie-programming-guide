# L2からの分散

[distribute_L2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/04_distribute_L2/distribute_L2.py)のデザインは、Object FIFO `of_in`を使用して外部メモリからL2に`24xi32`テンソルとしてデータを持ち込みます。そこから、データはより小さな`8xi32`部分で3つのObject FIFOに分散されます。各Workerは、アクセスする3つのObject FIFOのどれであるかに基づいて、より大きなデータの異なる部分を受け取ります。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/DistributeL2.svg" height=200 width="700">

```python
# ObjectFifosを使用したデータフロー
# 入力
of_offsets = [8 * worker for worker in range(n_workers)]

of_in = ObjectFifo(tile24_ty, name="in")
of_ins = (
    of_in
    .cons()
    .split(
        of_offsets,
        obj_types=[tile8_ty] * n_workers,
        names=[f"in{worker}" for worker in range(n_workers)],
    )
)
```

すべてのWorkerは、それぞれの入力Object FIFOから消費するために1つのオブジェクトをacquireし、そのすべてのエントリに`1`を追加し、オブジェクトをreleaseするという同じプロセスを実行しています。[join design](../05_join_L2/index.html)は、データが外部メモリに送り返され、テストされる方法を示しています。

このデザインをコンパイルするには、次のコマンドを使用できます：

```bash
make
```

このデザインの明示的に配置されたレベルのIRONプログラミングは、[distribute_L2_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/04_distribute_L2/distribute_L2_placed.py)で利用できます。次のコマンドでコンパイルできます：

```bash
env use_placed=1 make
```

このデータ移動パターンを含む他の例は、[programming_examples/matrix_multiplication/](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/)で利用できます。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/04_distribute_L2)を参照してください。
