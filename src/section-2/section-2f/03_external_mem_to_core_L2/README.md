# L2を経由した外部メモリからコアへ

[ext_to_coreL2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/03_external_mem_to_core_L2/ext_to_core.py)のデザインは、前の[例](../02_external_mem_to_core/index.html)と非常に似ていますが、このデザインでは最初に`of_in0`を使用して外部メモリからL2メモリ（つまりMemタイル）に`24xi32`データを持ち込む点が異なります。次に、`of_in1`を使用して、`MemTile`から`my_worker`にデータのより小さな`8xi32`スライスを持ち込みます。2つのFIFOが、最初に`of_out1`を介してL2に`8xi32`テンソルとしてデータを持ち込み、次に`of_out0`を介して外部メモリに`24xi32`テンソルとして持ち込みます。すべてのFIFOはダブルバッファを使用します。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/ExtMemToCoreL2.svg" height=200 width="500">

```python
# ObjectFifosを使用したデータフロー
# 入力
of_in0 = ObjectFifo(tile24_ty, name="in0")
of_in1 = of_in0.cons().forward(name="in1", obj_type=tile8_ty)

# 出力
of_out1 = ObjectFifo(tile8_ty, name="out1")
of_out0 = of_out1.cons().forward(name="out0", obj_type=tile24_ty)
```

Worker上のプロセスは、前のデザインと同じです。プロデューサプロセスは、消費するために`of_in1`から1つのオブジェクトをacquireし、生成するために`of_out1`から1つのオブジェクトをacquireします。次に、入力オブジェクトの値を読み取り、両方のオブジェクトをreleaseする前に、そのすべてのエントリに`1`を追加します。

このデザインをコンパイル、実行、テストするには、次のコマンドを使用できます：

```bash
make
make run
```

このデザインの明示的に配置されたレベルのIRONプログラミングは、[ext_to_core_L2_placed.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/03_external_mem_to_core_L2/ext_to_core_L2_placed.py)で利用できます。次のコマンドでコンパイル、実行、テストできます：

```bash
env use_placed=1 make
make run
```

[test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/03_external_mem_to_core_L2/test.cpp)およびデザインコードの`# To/from AIE-array data movement`セクションについては、[セクション2d](../../section-2d/index.html)で詳しく説明されます。

このデータ移動パターンを含む他の例は、[programming_examples/matrix_multiplication/](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/)で利用できます。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/03_external_mem_to_core_L2)を参照してください。
