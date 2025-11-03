# IRON AIE Programming Guide（非公式日本語訳）

本書は、[Xilinx/AMD mlir-aie](https://github.com/Xilinx/mlir-aie)プロジェクトの[Programming Guide](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide)を日本語に翻訳した非公式ガイドです。

## IRON（Interface Representation for hands-ON programming）について

**IRON**は、PythonバインディングとMLIRを使用したAI Engineアレイ開発のためのツールキットです。

AI Engineは**空間演算アーキテクチャ（spatial compute architecture）**です。空間的に分散された演算ユニットとメモリを持つ、モジュラーでスケーラブルなシステムです。AIE配列のプログラミングには、演算コア、データムーバー、相互接続ネットワークの設定が含まれます。

## ガイドの構成

本プログラミングガイドは以下のセクションで構成されています：

- **[Section 0](./section-0/README.md)**: ハードウェアセットアップと環境設定
- **[Section 1](./section-1/README.md)**: 基本的なAIE構成要素とPythonバインディング
- **[Section 2](./section-2/README.md)**: ObjectFIFOを使用したデータ移動の概念
- **[Section 3](./section-3/README.md)**: 入門用のベクトルスカラー乗算プログラム
- **[Section 4](./section-4/README.md)**: 性能測定とベクトルプログラミング技術
- **[Section 5](./section-5/README.md)**: ベクトル設計の例（パススルー、指数関数、GEMM、畳み込み）
- **[Section 6](./section-6/README.md)**: マルチコア性能を示す大規模設計

## 追加リソース

- AIEアーキテクチャマニュアル
  - [AM009](https://docs.amd.com/r/en-US/am009-versal-ai-engine)（AIE1用）
  - [AM020](https://docs.amd.com/r/en-US/am020-versal-aie-ml)（AIE2/AIE-ML用）
- [AMD XDNA NPU技術](https://www.amd.com/en/products/software/ai-accelerators.html)（Ryzen AIプロセッサ搭載）
- [クイックリファレンス](./quick_reference.md) - 重要なAPI要素のまとめ

## ライセンス

本翻訳は非公式のものです。元のドキュメントは Apache License v2.0 with LLVM Exceptions の下でライセンスされています。

---

**注意**: このガイドは学習と理解を助けるための非公式翻訳です。正確な情報については、[公式の英語版ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide)を参照してください。
