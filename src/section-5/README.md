# Section 5 - Example Vector Designs

このセクションでは、Ryzen™ AIのNPU配列内でAI Engineの機能を実証するプログラミング例を紹介します。

## 最もシンプルな例

基礎的な例には、**Passthrough**と**Passthrough DMAs**が含まれます。Passthrough設計は、「ベクトル化されたロードとストアを使用して、入力から出力へ4096バイトをコピー」し、4つの必須ファイルで構成されます：Python構造設計、C++カーネル実装、テストアプリケーション、Makefile。

## 基本設計

15の基礎的な例が、整数および浮動小数点演算にまたがります：

### 算術演算
- スカラー加算/ベクトル加算
- スカラー乗算/ベクトル乗算
- 剰余演算

### リダクション関数
- 総和（sum）
- 最大値（max）
- 最小値（min）

### 数学演算
- 指数関数（exponential function）

### 行列演算
- DMA転置
- 単一コア/マルチコアGEMM（General Matrix Multiply）
- GEMV（General Matrix-Vector multiply）

使用されるデータ型：`i32`、`bfloat16`、`i8`

## 機械学習カーネル

ニューラルネットワーク開発をサポートする6つの特化した例：

### 要素ごとの演算
- 加算（add）
- 乗算（multiply）

### 活性化関数
- ReLU
- Softmax

### 畳み込み層
- Conv2D（オプションでReLU融合）

## 学習演習

より深い探求を促す4つの段階的な課題：

1. **設計の修正**: 既存の設計を変更して理解を深める
2. **データ型分析**: 異なるデータ型の影響を調査
3. **性能メトリクス**: 通信対計算比の測定
4. **カーネルコンポーネント識別**: より大きな演算内のカーネル要素の特定

## プロジェクト構造

各例は通常、以下のファイルで構成されます：

- **Python構造設計**: AIE配列の設定
- **C++カーネル実装**: 実際の計算ロジック
- **テストアプリケーション**: ホストコードと検証
- **Makefile**: ビルドとテストの自動化

---

**注意**: 各設計例の完全なソースコードと詳細な説明については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-5)を参照してください。

-----

[[前へ - Section 4](../section-4/README.md)] [[トップ](../README.md)] [[次へ - Section 6](../section-6/README.md)]
