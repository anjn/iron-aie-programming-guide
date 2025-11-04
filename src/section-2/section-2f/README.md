# Section 2f - 実践的な例

このセクションでは、一般的なObject FIFOデータ移動パターンを含むいくつかの例を紹介します。これらの例は、他の設計に簡単にインポートして適応できるように十分にシンプルに設計されています。

## [例1 - シングル/ダブルバッファ](01_single_double_buffer/index.html)

シングル/ダブルバッファを使用したコア間のデータ移動

## [例2 - 外部メモリからコアへ](02_external_mem_to_core/index.html)

ダブルバッファを使用した外部メモリとコア間の往復データ移動

## [例3 - L2を経由した外部メモリからコアへ](03_external_mem_to_core_L2/index.html)

ダブルバッファを使用してL2を経由した外部メモリとコア間の往復データ移動

## [例4 - L2からの分散](04_distribute_L2/index.html)

L2を通じて外部メモリから複数のコアにデータを分散

## [例5 - L2での結合](05_join_L2/index.html)

L2を通じて複数のコアから外部メモリにデータを結合

---

**注意**: 各例の完全なソースコードと詳細な説明については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2f)を参照してください。
