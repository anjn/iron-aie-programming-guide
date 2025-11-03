# Section 2f - 実践的な例

このセクションでは、一般的なObject FIFOデータ移動パターンを含むいくつかの例を紹介します。これらの例は、他の設計に簡単にインポートして適応できるように十分にシンプルに設計されています。

## 例1 - シングル/ダブルバッファ

[Example 01 - Single / Double Buffer](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/01_single_double_buffer/)

* シングル/ダブルバッファを使用したコア間のデータ移動

## 例2 - 外部メモリからコアへ

[Example 02 - External Memory to Core](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/02_external_mem_to_core/)

* ダブルバッファを使用した外部メモリとコア間の往復データ移動

## 例3 - L2を経由した外部メモリからコアへ

[Example 03 - External Memory to Core through L2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/03_external_mem_to_core_L2/)

* ダブルバッファを使用してL2を経由した外部メモリとコア間の往復データ移動

## 例4 - L2からの分散

[Example 04 - Distribute from L2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/04_distribute_L2/)

* L2を通じて外部メモリから複数のコアにデータを分散

## 例5 - L2での結合

[Example 05 - Join in L2](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f/05_join_L2/)

* L2を通じて複数のコアから外部メモリにデータを結合

---

**注意**: 各例の完全なソースコードと詳細な説明については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f)を参照してください。
