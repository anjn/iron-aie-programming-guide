# Section 2g - Object FIFOを使わないデータ移動

## 概要

この高度なガイドセクションは、Object FIFOが必要なパターンを記述できない場合に、AIEタイルでのデータ移動にDirect Memory Access（DMA）チャネルを使用することに焦点を当てています。この資料は**「IRONレベル」**の難易度としてマークされています。

## タイルタイプとDMA初期化

AIEアーキテクチャには3つのタイルタイプがあります：

### 1. コンピュートタイル（Compute tiles）

**宣言:** `tile(col, row)`
**初期化:** `@mem(tile)`

**特徴:**
- 2つの入力ポート
- 2つの出力ポート
- L1メモリ付き

### 2. メモリタイル（Memory tiles）

**宣言:** `tile(col, row)` （通常は行1）
**初期化:** `@memtile_dma(tile)`

**特徴:**
- 6つの入力ポート
- 6つの出力ポート
- 大容量L2メモリ

### 3. シムタイル（Shim tiles）

**宣言:** `tile(col, row)` （通常は行0）
**初期化:** `@shim_dma(tile)`

**特徴:**
- 2つの入力ポート
- 2つの出力ポート
- 外部メモリへのアクセス

> **注意:** すべてのタイルタイプは同じ基本DMA設計を使用します。

## DMAチャネルの設定

### チャネル方向の分類

チャネルは方向によって分類されます：

- **S2MM（Slave-to-Memory Mapped）**: 入力チャネル（ストリームからメモリへ）
- **MM2S（Memory Mapped-to-Slave）**: 出力チャネル（メモリからストリームへ）

### 統一された`dma`コンストラクタ

```python
dma(
    channel_dir,        # S2MM または MM2S
    channel_index,      # チャネル番号
    num_blocks=1,       # バッファディスクリプタ数
    loop=None,          # ループ設定
    repeat_count=None,  # 繰り返し回数
    sym_name=None,      # シンボル名
    loc=None,           # 位置
    ip=None             # IPアドレス
)
```

## バッファディスクリプタ（BDs）

### 基本概念

データ移動は、**バッファディスクリプタのチェーン**を通じて記述されます。各BDは以下を指定します：

- 移動するデータ
- 同期の設定
- アドレスパターン

### BDの追加

追加のBDは、`@another_bd(dma_bd)`デコレータを使用して追加されます。

```python
@dma(S2MM, channel_index=0)
def dma_handler():
    # 最初のBD
    buffer_descriptor(buffer1, ...)

    @another_bd(dma_handler)
    def second_bd():
        # 2番目のBD
        buffer_descriptor(buffer2, ...)
```

## ロックによる同期

### ロックの役割

ドキュメントでは以下のように説明されています：「タイルが現在`buff_in`を使用している場合、それが完了したときのみ`prod_lock`を解放し、その時にDMAがデータを上書きすることが許可されます。」

### ロックの使用パターン

ロックは、タイル実行とDMA操作間の同期ポイントをマークし、プロデューサ-コンシューマパターンを可能にします。

```python
# プロデューサ側
use_lock(prod_lock, AcquireGreaterEqual, value=0)  # 空きバッファを待つ
# データ書き込み
use_lock(prod_lock, Release, value=1)              # データ準備完了を通知

# コンシューマ側
use_lock(cons_lock, AcquireGreaterEqual, value=1)  # データ準備を待つ
# データ読み取り
use_lock(cons_lock, Release, value=0)              # バッファ解放を通知
```

## ダブルバッファリングパターン

### ピンポンバッファリング

`num_blocks=2`と増加したロックトークン数を使用して、交互のバッファ使用パターンを作成します。

```python
@dma(S2MM, channel_index=0, num_blocks=2)
def double_buffer_dma():
    # バッファ1用のBD
    use_lock(lock, AcquireGreaterEqual, value=0)
    buffer_descriptor(buffer1, ...)
    use_lock(lock, Release, value=2)  # トークン数を2に

    # バッファ2用のBD
    @another_bd(double_buffer_dma)
    def bd2():
        use_lock(lock, AcquireGreaterEqual, value=0)
        buffer_descriptor(buffer2, ...)
        use_lock(lock, Release, value=2)
```

**動作:**
1. バッファ1に書き込み中、バッファ2を読み取り可能
2. バッファ2に書き込み中、バッファ1を読み取り可能
3. 並行性の向上

## タイル間のデータフロー

### `flow`コンストラクタ

異なるタイル上のDMAチャネル間の接続を確立します。

```python
flow(
    source_tile,           # ソースタイル
    WireBundle.DMA,        # バンドル指定
    source_channel,        # ソースチャネルインデックス
    dest_tile,             # 宛先タイル
    WireBundle.DMA,        # バンドル指定
    dest_channel           # 宛先チャネルインデックス
)
```

**注意:** 「フロー低レベル化は、`source`と`dest`入力に基づいて方向を推論できます。」

## 完全な2タイル例

### シナリオ

1つのタイルがMM2S（出力チャネル0）を介してデータを送信し、別のタイルのS2MM（入力チャネル1）に送ります。

### プロデューサタイル

```python
@mem(producer_tile)
def producer_mem():
    @dma(MM2S, channel_index=0)
    def send_dma():
        use_lock(send_lock, AcquireGreaterEqual, value=1)
        buffer_descriptor(send_buffer, length=data_size)
        use_lock(send_lock, Release, value=0)
```

### コンシューマタイル

```python
@mem(consumer_tile)
def consumer_mem():
    @dma(S2MM, channel_index=1)
    def receive_dma():
        use_lock(recv_lock, AcquireGreaterEqual, value=0)
        buffer_descriptor(recv_buffer, length=data_size)
        use_lock(recv_lock, Release, value=1)
```

### 接続

```python
flow(
    producer_tile, WireBundle.DMA, 0,  # ソース: MM2S チャネル0
    consumer_tile, WireBundle.DMA, 1   # 宛先: S2MM チャネル1
)
```

## Object FIFOとの比較

### Object FIFOを使用する場合

- 標準的なデータフローパターン
- 高レベル抽象化が望ましい
- 迅速な開発

### 低レベルDMAを使用する場合

- Object FIFOで表現できないパターン
- 細かい制御が必要
- カスタムDMA設定
- 性能の最大限の最適化

## 高度なDMA機能

### n次元転送

バッファディスクリプタは、n次元アドレスパターンをサポート：

```python
buffer_descriptor(
    buffer,
    offset=0,
    dimensions=[[size_2, stride_2], [size_1, stride_1], [size_0, stride_0]]
)
```

### パケット化

パケットベースのルーティングにより、柔軟なデータフロー：

```python
use_packet(packet_id, packet_type)
```

## デバッグのヒント

### 一般的な問題

1. **デッドロック**: ロックの取得/解放の不一致
2. **データ破損**: 不適切な同期
3. **性能の問題**: 非効率的なバッファリング

### デバッグ戦略

- ロックの状態を追跡
- DMAチャネルのステータスを確認
- トレース機能を使用
- 段階的にテスト

---

**注意**: この高度なトピックの完全な実装例と詳細については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2g)を参照してください。

-----

[[前へ - Section 2f](../section-2f/README.md)] [[トップ](../../README.md)]
