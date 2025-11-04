# 外部メモリからコアへ

[ext_to_core.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/02_external_mem_to_core/ext_to_core.py)のデザインは、Object FIFO `of_in`を使用して外部メモリから`my_worker`にデータを持ち込み、別のObject FIFO `of_out`を使用してWorkerから外部メモリにデータを送信します。各FIFOはダブルバッファを使用します。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/ExtMemToCore.svg" height=200 width="400">

```python
# ObjectFifosを使用したデータフロー
of_in = ObjectFifo(tile_ty, name="in")
of_out = ObjectFifo(tile_ty, name="out")
```

コンシューマとプロデューサの両方のプロセスが`my_worker`で実行されています。プロデューサプロセスは、消費するために`of_in`から1つのオブジェクトをacquireし、生成するために`of_out`から1つのオブジェクトをacquireします。次に、入力オブジェクトの値を読み取り、両方のオブジェクトをreleaseする前に、そのすべてのエントリに`1`を追加します。

このデザインをコンパイル、実行、テストするには、次のコマンドを使用できます：

```bash
make
make run
```

このデザインの明示的に配置されたレベルのIRONプログラミングは、[ext_to_core_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/02_external_mem_to_core/ext_to_core_placed.py)で利用できます。次のコマンドでコンパイル、実行、テストできます：

```bash
env use_placed=1 make
make run
```

[test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/02_external_mem_to_core/test.cpp)およびデザインコードの`# To/from AIE-array data movement`セクションについては、[セクション2d](../../section-2d/index.html)で詳しく説明されます。

このデータ移動パターンを含む他の例は、[programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)で利用できます。注目すべき例としては、[vector_reduce_add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_reduce_add/)と[vector_scalar_add](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_add/)があります。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/02_external_mem_to_core)を参照してください。
