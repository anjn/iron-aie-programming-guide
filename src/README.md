# IRON AIE Programming Guide（非公式日本語訳）

<img align="right" width="300" height="300" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/main/programming_guide/assets/AIEarray.svg">

AI Engine（AIE）アレイは、**空間演算アーキテクチャ（spatial compute architecture）**です。空間的に分散された演算ユニットとメモリを持つ、モジュラーでスケーラブルなシステムです。その演算密度の高いベクトル処理は、明示的にスケジュールされたデータ移動と独立して並行実行されます。各AIEのベクトル演算コア（緑色）は、そのL1スクラッチパッドメモリ（水色）内のデータに対してのみ動作できるため、Direct Memory Access（DMA）チャネル（紫色）が、メモリ階層の任意のレベルから、スイッチ型（濃青色）相互接続ネットワークを介して双方向にデータを転送します。

AIE配列のプログラミングでは、すべての空間的構成要素を設定します：演算コアのプログラムメモリ、データムーバーのバッファディスクリプタ、スイッチを含む相互接続など。本ガイドでは、AIE配列の**IRON**（Interface Representation for hands-ON）プログラミングを紹介します。IRONは、mlir-aie（AIE配列のMLIRベース表現）を中心とした一連のPython言語バインディングを通じて、パフォーマンスエンジニアが高速で効率的な、しばしば特化した設計を構築できるようにするオープンアクセスツールキットです。mlir-aieは、複雑で高性能なAI Engine設計を定義できる基盤を提供し、シミュレーションとハードウェア実装インフラストラクチャによってサポートされています。

IRONは、ユーザーの経験レベルに合わせてAIE配列のプログラミングへの複数のエントリーポイントを提供します。最も高い抽象化レベルでは、基盤となるハードウェアアーキテクチャの深い知識を必要とせずに、専用タスクをワーカーに割り当てるプログラムを作成できます。AIE配列の設定をより細かく制御したいユーザーには、IRONは明示的な配置を行うAPIをサポートしています。本ガイドは、両方のプログラミングレベルが各セクションで説明されるように構成されています。

> **注意**: NPUをIRONでプログラミングする方法を素早く理解したい方は、[ミニチュートリアル](mini_tutorial/index.html)をご覧ください！

このIRON AIEプログラミングガイドでは、まずAIE配列の構造要素に対する言語バインディングを紹介します（[セクション1](section-1/index.html)）。必要なデータを転送するための明示的なデータ移動の設定方法を説明した後（[セクション2](section-2/index.html)）、AIE演算コアで最初のプログラムを実行できます（[セクション3](section-3/index.html)）。[セクション4](section-4/index.html)では、性能分析のためのトレース機能を追加し、演算密度の高いベクトル演算の活用方法を説明します。基本的なものから大規模なもの（機械学習やコンピュータビジョン）まで、より多くのベクトル設計例をセクション[5](section-5/index.html)と[6](section-6/index.html)で紹介します。最後に、[クイックリファレンス](quick_reference.html)で最も重要なAPI要素をまとめています。

## 目次

<details><summary><a href="section-0/index.html">Section 0 - Getting Set Up for IRON</a></summary>

* IRONでターゲットとする推奨ハードウェアの紹介
* ハードウェア、ツール、環境のセットアップ手順
</details>

<details><summary><a href="section-1/index.html">Section 1 - Basic AI Engine building blocks</a></summary>

* アプリケーション設計を表現するためのAI Engine構成要素の紹介
* AIEタイルを定義するMLIRソースのPythonバインディング例
</details>

<details><summary><a href="section-2/index.html">Section 2 - Data Movement (Object FIFOs)</a></summary>

* objectfifoとそれがタイル間の接続とAIE配列メモリ内のデータをどのように抽象化するかを紹介
* 主要なobjectfifoデータ移動パターンの説明
* より複雑なobjectfifo接続パターン（broadcast、implicit copy、join、distribute）の紹介
* 実践的な例でobjectfifoを実演
* ホストとAIE配列間のランタイムデータ移動の説明
</details>

<details><summary><a href="section-3/index.html">Section 3 - My First Program</a></summary>

* 最初のシンプルなプログラム（ベクトルスカラー乗算）の例を紹介
* Ryzen™ AI対応ハードウェアで設計を実行する方法を説明
</details>

<details><summary><a href="section-4/index.html">Section 4 - Performance Measurement & Vector Programming</a></summary>

* 性能測定（タイマー、トレース）の紹介
* カーネルレベルでのベクトルプログラミングの説明
</details>

<details><summary><a href="section-5/index.html">Section 5 - Example Vector Designs</a></summary>

* 性能測定の演習を含む追加のベクトル設計例：
    * パススルー
    * ベクトル $e^x$
    * ベクトルスカラー加算
    * GEMM
    * CONV2D
    * その他
</details>

<details><summary><a href="section-6/index.html">Section 6 - Larger Example Designs</a></summary>

* 複数のコアで性能を測定した大規模設計例：
    * エッジ検出
    * ResNet
    * その他
</details>

### [クイックリファレンス](quick_reference.html)

## AI Engineアーキテクチャドキュメント
* [AIE1 Architecture Manual - AM009](https://docs.amd.com/r/en-US/am009-versal-ai-engine/Overview)
* [AIE2 Architecture Manual - AM020](https://docs.amd.com/r/en-US/am020-versal-aie-ml/Overview)

## AMD XDNA™ リファレンス
* [AMD XDNA™ NPU in Ryzen™ AI Processors](https://ieeexplore.ieee.org/document/10592049)

---

**注意**: これは[公式プログラミングガイド](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide)の非公式日本語訳です。正確な情報については、常に公式の英語版ドキュメントを参照してください。

**ライセンス**: 元のドキュメントは Apache License v2.0 with LLVM Exceptions の下でライセンスされています。
