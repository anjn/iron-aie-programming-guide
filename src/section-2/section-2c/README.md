# Section 2c - データレイアウト変換

## 概要

このセクションでは、「タイルの専用ハードウェアを使用したオンザフライのデータレイアウト変換：Direct Memory Access（DMA）チャネル」について説明します。この機能は**AIE-MLデバイスでのみ利用可能**です。

## タイルDMAの役割

タイルDMAは、ローカルメモリとAXIストリーム相互接続の間でデータを移動しながら、プログラマブルなn次元アドレス生成スキームを可能にします。

## バッファディスクリプタ

### 基本概念

MLIRでのバッファディスクリプタ操作（`AIE_DMABDOp`）は、以下を制御します：

- **何を**移動するか
- **どこから**（オフセット）
- **どれだけ**（量）
- **どのように**（レイアウトパターン）

### Pythonバインディングパラメータ

- **buffer**: データバッファ
- **offset**: 開始位置
- **length**: データ量
- **dimensions**: 次元情報
- **buffer descriptor IDs**: ディスクリプタID

## 変換の表現

### 次元とストライドの指定

変換は、各次元のサイズとストライドのタプルペアで表現されます：

```
[<size_2, stride_2>, <size_1, stride_1>, <size_0, stride_0>]
```

### サポートされる次元数

- **コンピュートタイルとシムタイル**: 最大3次元
- **メモリタイル**: 最大4次元

### 重要な制約

「**すべてのストライドは要素幅の倍数で表現されます。**」

**重要**: 「**4Bデータ型の場合のみ、最内次元のストライドは設計上1でなければなりません。**」

## アクセスパターンモデル

### ネストされたループとしての理解

変換は、順次データアクセスを決定するネストされたループとして機能します：

```python
# 擬似コード
for i2 in range(size_2):  # 最外次元、stride_2を使用
    for i1 in range(size_1):  # 中間次元、stride_1を使用
        for i0 in range(size_0):  # 最内次元、stride_0を使用
            # アクセス: base + i2*stride_2 + i1*stride_1 + i0*stride_0
```

### 実践例

**シナリオ**: 128要素のバッファから、8要素ごとに交互に偶数要素と奇数要素にアクセス

```python
# 偶数要素の取得
dimensions_even = [[4], [2], [8, 2]]
# size=8, stride=2 → 各グループで8要素を2つおきに
# size=2 → 2グループ
# size=4 → 4回繰り返し

# 対応するネストループ
for i in range(4):
    for j in range(2):
        for k in range(8):
            index = i*16 + j*2 + k*2
            # アクセス
```

## Object FIFOとの統合

### コンストラクタパラメータ

`object_fifo`クラスコンストラクタは、以下のパラメータでデータレイアウト変換を可能にします：

- **`dimensionsToStream`**: プロデューサ側DMAに、データをどの順序でプッシュするかを指示
- **`dimensionsFromStreamPerConsumer`**: コンシューマ側DMAに、ストリームからどのレイアウトでデータを取得するかを指示

### 具体例

`<4x8xi8>`オブジェクトの変換で、偶数行から特定要素を選択：

```python
object_fifo(
    "fifo_name",
    producer_tile,
    consumer_tile,
    depth=2,
    dtype=tensor_4x8_i8,
    dimensionsToStream=[[2], [4], [2, 8]],      # プロデューサ側
    dimensionsFromStreamPerConsumer=[[1], [2]]  # コンシューマ側
)
```

## ランタイムシーケンスでの使用

### `fill()`と`drain()`操作

「ランタイムシーケンス操作（`fill()`や`drain()`など）は、オプションで`tap`を入力として受け取り、外部メモリとの間のアクセスパターンをオンザフライで変更できます。」

### `taplib`ライブラリ

`taplib`ライブラリは、IRONを使用したAIE用のテンソルアクセスパターン機能を提供します。

## 実装例

### 行列転置

128x128行列の転置をDMAを使用して実行：

```python
# 入力: 行メジャー
dimensionsToStream=[[128], [128]]

# 出力: 列メジャー
dimensionsFromStream=[[128], [128, 128]]  # stride=128で列アクセス
```

### 選択的データ抽出

特定のパターンでデータを抽出：

```python
# 4x4行列から対角要素のみ抽出
dimensionsFromStream=[[4, 5]]  # size=4, stride=5（次の対角へ）
```

## 参考実装

公式リポジトリの`programming_examples`ディレクトリには、これらの変換技術を利用した注目すべき実装があります：

- 行列演算の例
- 転置実装
- データ再配置パターン

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2c)を参照してください。

-----

[[前へ - Section 2b](../section-2b/README.md)] [[次へ - Section 2d](../section-2d/README.md)] [[トップ](../../README.md)]
