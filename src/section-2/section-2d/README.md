# Section 2d - ランタイムデータ移動

## 概要

このセクションでは、NPUデバイス上のAI Engine（AIE）配列におけるランタイムデータ移動のフレームワークを確立します。前のセクションではAIE配列内部のデータ移動を扱いましたが、実用的なアプリケーションでは、外部ソース（「ホスト」）とAIE配列間の情報転送が必要です。

## 核心概念

実際のアプリケーションでは、以下のデータ移動が必要です：

- **ホスト → AIE配列**: 入力データの転送
- **AIE配列 → ホスト**: 処理結果の転送

この機能は、NPUデバイス上で特定の操作を通じて実現されます。

## 技術要件

### ランタイムシーケンスの配置

ホストと配列間の通信のための操作は、以下のいずれかに配置する必要があります：

1. **高レベルIRON**: `Runtime`クラス内の専用`sequence()`関数
2. **明示的配置IRON**: `aie.runtime_sequence`操作

### 関数構造

```python
# 高レベルIRONの例
rt = Runtime()
with rt.sequence(input_type, output_type) as (inp, out):
    # データ移動操作
    rt.fill(of_in, inp)
    rt.drain(of_out, out)
```

**ポイント:**
- 関数引数は、ホスト側からアクセス可能なバッファを表す
- 関数本体は、移動メカニズムを指定する

## アプローチの選択

このガイドでは、ユーザーのニーズに応じた抽象化レベルに対応する2つの異なるアプローチを提供します：

### 1. 高レベル構造（推奨）

**参照**: RuntimeTasks リファレンス

**特徴:**
- より抽象的で使いやすい
- 詳細な設定が隠蔽される
- 初心者に適している

**主な操作:**
- `rt.fill()`: ホストからAIE配列へデータ転送
- `rt.drain()`: AIE配列からホストへデータ転送
- `rt.start()`: Workerの開始

### 2. 低レベル関数

**参照**: DMATasks ドキュメント

**主な関数:**
- `npu_dma_memcpy_nd()`: n次元DMAメモリコピー
- `dma_wait()`: DMA完了待機

**特徴:**
- より細かい制御が可能
- ハードウェアに近い操作
- 高度なユーザー向け

## 実践例

### 基本的なデータ転送

```python
# Runtime インスタンスの作成
rt = Runtime()

# ランタイムシーケンスの定義
with rt.sequence(data_type, data_type, data_type) as (inp, factor, out):
    # 入力データをAIE配列に転送
    rt.fill(of_in, inp)

    # スカラーファクターを転送
    rt.fill(of_factor, factor)

    # Workerを開始
    rt.start(my_worker)

    # 結果をホストに転送
    rt.drain(of_out, out)
```

### n次元DMA転送（低レベル）

```python
# 2D配列の転送例
npu_dma_memcpy_nd(
    buffer=output_buffer,
    offset=0,
    sizes=[HEIGHT, WIDTH],
    strides=[WIDTH, 1]
)

# 完了を待機
dma_wait(of_output)
```

## データフローの同期

### 同期の必要性

ホストとAIE配列間のデータ転送は、以下を確保するために同期が必要です：

- データの整合性
- 処理の順序保証
- リソースの競合回避

### 同期メカニズム

```python
# Object FIFOを使用した暗黙的同期
rt.fill(of_in, input_data)   # 転送が完了するまで待機
rt.start(worker)              # データ準備後に開始
rt.drain(of_out, output_data) # 処理完了後に転送

# 明示的な待機
dma_wait(of_in, of_out)  # 複数のObject FIFOを待機
```

## バッファ管理

### ホスト側バッファ

- XRTバッファオブジェクトとして作成
- ホストメモリとデバイスメモリ間でマッピング
- `group_id`による識別

### AIE側バッファ

- Object FIFOによって管理
- タイルのローカルメモリに配置
- DMAチャネルを通じてアクセス

## 性能最適化

### バッファリング戦略

- **ダブルバッファリング**: データ転送と処理の並行化
- **深いFIFO**: より多くのバッファリングで帯域幅を最大化

### DMA設定の最適化

- **バースト転送**: 連続データの効率的な転送
- **n次元転送**: 複雑なデータレイアウトの直接サポート

---

**注意**: より詳細な情報とAPI リファレンスについては、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2d)を参照してください。

-----

[[前へ - Section 2c](../section-2c/README.md)] [[次へ - Section 2e](../section-2e/README.md)] [[トップ](../../README.md)]
