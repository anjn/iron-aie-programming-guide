# Summary

[はじめに](./README.md)

---

- [Section 0 - IRON環境構築](./section-0/README.md)

- [Section 1 - AI Engineの基本構成要素](./section-1/README.md)

- [Section 2 - データ移動（Object FIFO）](./section-2/README.md)
  - [2a - 基礎](./section-2/section-2a/README.md)
  - [2b - 通信パターン](./section-2/section-2b/README.md)
    - [2b-1 - 再利用パターン](./section-2/section-2b/01_Reuse/README.md)
    - [2b-2 - ブロードキャストパターン](./section-2/section-2b/02_Broadcast/README.md)
    - [2b-3 - 暗黙的コピー（Distribute/Join）](./section-2/section-2b/03_Implicit_Copy/README.md)
    - [2b-4 - リピートパターン](./section-2/section-2b/04_Repeat/README.md)
  - [2c - データレイアウト変換](./section-2/section-2c/README.md)
  - [2d - ランタイムデータ移動](./section-2/section-2d/README.md)
    - [2d-1 - RuntimeTasks](./section-2/section-2d/RuntimeTasks.md)
    - [2d-2 - DMATasks](./section-2/section-2d/DMATasks.md)
  - [2e - マルチコア設計](./section-2/section-2e/README.md)
  - [2f - 実践的な例](./section-2/section-2f/README.md)
    - [2f-1 - シングル/ダブルバッファ](./section-2/section-2f/01_single_double_buffer/README.md)
    - [2f-2 - 外部メモリからコアへ](./section-2/section-2f/02_external_mem_to_core/README.md)
    - [2f-3 - L2経由で外部メモリからコアへ](./section-2/section-2f/03_external_mem_to_core_L2/README.md)
    - [2f-4 - L2からの分散](./section-2/section-2f/04_distribute_L2/README.md)
    - [2f-5 - L2での結合](./section-2/section-2f/05_join_L2/README.md)
  - [2g - DMAプログラミング](./section-2/section-2g/README.md)

- [Section 3 - はじめてのプログラム](./section-3/README.md)

- [Section 4 - 性能測定とベクトルプログラミング](./section-4/README.md)
  - [4a - タイマー](./section-4/section-4a/README.md)
  - [4b - トレース](./section-4/section-4b/README.md)
  - [4c - カーネルのベクトル化と最適化](./section-4/section-4c/README.md)

- [Section 5 - ベクトル設計の例](./section-5/README.md)

- [Section 6 - 大規模設計の例](./section-6/README.md)

---

[ミニチュートリアル](./mini_tutorial/README.md)
[クイックリファレンス](./quick_reference.md)
