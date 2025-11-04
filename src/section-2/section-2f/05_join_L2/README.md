# L2での結合

[join_L2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/join_L2.py)のデザインには3つのWorkerがあり、それぞれが3つのObject FIFOのうちの1つを介してL2に`8xi32`のデータを送信します。そこで、データは各Object FIFOのオフセットに基づいて`24xi32`テンソルに結合されます。次に、`of_out`を使用してデータが外部メモリに送信されます。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/JoinL2.svg" height=200 width="700">

```python
# ObjectFifosを使用したデータフロー
# 出力
of_offsets = [8 * worker for worker in range(n_workers)]

of_out = ObjectFifo(tile24_ty, name="out")
of_outs = (
    of_out.prod().join(
        of_offsets,
        obj_types=[tile8_ty] * n_workers,
        names=[f"out{worker}" for worker in range(n_workers)],
    )
)
```

すべてのWorkerは、それぞれの入力Object FIFOから生成するために1つのオブジェクトをacquireし、そのすべてのエントリに`1`を書き込み、オブジェクトをreleaseするという同じプロセスを実行しています。

このデザインは、前の[distribute](../04_distribute_L2/index.html)デザインと組み合わされて、外部メモリからAIE配列への完全なデータ移動とその逆を実現します。結果のコードは、[distribute_and_join_L2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/distribute_and_join_L2.py)で利用できます。次のコマンドでコンパイル、実行、テストできます：

```bash
make
make run
```

このデザインの明示的に配置されたレベルのIRONプログラミングは、[distribute_and_join_L2_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/distribute_and_join_L2_placed.py)で利用できます。次のコマンドでコンパイル、実行、テストできます：

```bash
env use_placed=1 make
make run
```

[test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/test.cpp)およびデザインコードの`# To/from AIE-array data movement`セクションについては、[セクション2d](../../section-2d/index.html)で詳しく説明されます。

> **注意：** [distribute_and_join_L2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/distribute_and_join_L2.py)のデザインは、[ext_to_core](../03_external_mem_to_core_L2/index.html)を取り、入力データのより小さな部分を3つのWorkerに分散します。このパターンは通常、入力データが単一のコアのメモリモジュールに対して大きすぎて、より小さなチャンクで処理する必要がある場合に使用され、その結果が結合されて最終的な出力が生成されます。

このデータ移動パターンを含む他の例は、[programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)で利用できます。注目すべき例は[vector_exp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_exp/)です。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2)を参照してください。
