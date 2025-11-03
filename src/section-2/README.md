# Section 2 - Data Movement (Object FIFOs)

このガイドでは、AIE配列内のデータ移動を記述するための高レベル通信プリミティブである**Object FIFO**を紹介します。

## 主な学習目標

このガイドは、開発者が以下を習得することを目的としています：

1. 通信プリミティブAPIを概念レベルで理解する
2. 実践的な設計例を通じて初期化とアクセス方法を学ぶ
3. 現在のObject FIFOの制限の背後にある設計上の決定を理解する
4. 実装詳細に関するより深い資料を見つける

## 核心概念

「AIE配列は、明示的なデータ移動要件を持つ空間演算アーキテクチャです。」各処理ユニットはそのL1メモリ内のデータに対して動作しますが、これは配列全体での移動のために明示的に設定する必要があります。Object FIFOにより、開発者は高度なハードウェア制御機能を維持しながら、このデータ移動をアクセス可能な方法で指定できます。

## ガイドの構成

資料は7つの段階的なセクションで構成されています：

- **[Section 2a](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2a)**: 基礎（初期化、アクセス、同一プロデューサ/コンシューマシナリオ）
- **[Section 2b](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2b)**: 主要パターン（再利用、ブロードキャスト、分散、結合）
- **[Section 2c](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2c)**: データレイアウト変換
- **[Section 2d](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2d)**: ホストメモリとの間のランタイムデータ移動
- **[Section 2e](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2e)**: マルチコア設計の最適化
- **[Section 2f](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2f)**: 一般的な設計パターンを含む実践的なコード例
- **[Section 2g](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2g)**: 代替のDMAリージョンプログラミングアプローチ

## Object FIFOとは

Object FIFOは、AIE配列内の異なるタイル間でデータを移動するための抽象化レイヤーです。従来の手動DMA設定と比較して、以下の利点があります：

- **高レベル抽象化**: 複雑なDMA設定を隠蔽
- **明示的なデータフロー**: データの流れを明確に表現
- **柔軟な通信パターン**: ブロードキャスト、分散、結合などをサポート
- **効率的なバッファ管理**: 自動的なバッファリング戦略

---

**注意**: 各サブセクションの詳細な翻訳は進行中です。最新の情報は[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2)を参照してください。

-----

[[前へ - Section 1](../section-1/README.md)] [[トップ](../README.md)] [[次へ - Section 3](../section-3/README.md)]
