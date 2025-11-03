# Section 2e - マルチコアプログラミング

## 概要

このセクションでは、単一コアのコードをマルチコア設計に変換する方法を、2つのIRON APIレベルで実演します。プロセスは、1つのWorker/コンピュートタイルから3つへとスケーリングし、同じ基本的なロジックを維持しながら、処理ユニット間でワークロードを分散します。

## 高レベルIRONアプローチ

### 初期の単一コア設計

基盤となる設計は、`<48xi32>`型のデータオブジェクトを扱う4つのObject FIFOを使用します。

**アーキテクチャ:**

```
入力パス: 外部メモリ → メモリタイル → Worker
          (of_in)    (of_in0)

出力パス: Worker → メモリタイル → 外部メモリ
          (of_out0)    (of_out)
```

**Workerタスク:**
1. 入力オブジェクトを取得
2. 各エントリを1ずつインクリメント
3. 結果を出力にコピー
4. 両方を解放

### マルチコアへのスケーリング

変換では、単一のメモリタイルを維持しながら3つのWorkerに拡張します。各Workerは、`<16xi32>`オブジェクト（48÷3）を処理します。

#### 主な変更点

**1. データ移動設定:**

```python
# 入力: 分散操作を使用
# 完全なデータセットを3つのObject FIFOに分配
of_in0_fifos = [
    ObjectFifo(tile_size_type, name=f"in0_{i}", depth=2)
    for i in range(3)
]

# オフセットでデータ位置を追跡
offsets = [0, 16, 32]

# 出力: 結合操作を使用
# 3つの別々のFIFOから結果を収集
of_out0_fifos = [
    ObjectFifo(tile_size_type, name=f"out0_{i}", depth=2)
    for i in range(3)
]
```

**2. Worker作成:**

```python
# ループで複数のWorkerをインスタンス化
workers = []
for i in range(3):
    worker = Worker(
        core_fn,
        [of_in0_fifos[i].cons(), of_out0_fifos[i].prod()],
        name=f"worker_{i}"
    )
    workers.append(worker)
```

**3. ランタイムシーケンス:**

```python
# すべてのWorkerを同時に開始
rt.start(*workers)

# データフロー管理
rt.fill(of_in, input_data)
rt.drain(of_out, output_data)
```

## 明示的配置レベルIRON

### タイル宣言

**単一コア設定:**

```python
ShimTile = tile(0, 0)
MemTile = tile(0, 1)
ComputeTile = tile(0, 2)
```

**マルチコア設定:**

```python
ShimTile = tile(0, 0)
MemTile = tile(0, 1)
ComputeTile1 = tile(0, 2)
ComputeTile2 = tile(0, 3)
ComputeTile3 = tile(0, 4)
```

### データ移動アーキテクチャ

高レベルIRONと同様の分散/結合パターンを採用：

**入力Object FIFO:**

```python
# 完全データを3つのストリームに分割
of_in0_0 = object_fifo("in0_0", MemTile, ComputeTile1, 2, tile_size_type)
of_in0_1 = object_fifo("in0_1", MemTile, ComputeTile2, 2, tile_size_type)
of_in0_2 = object_fifo("in0_2", MemTile, ComputeTile3, 2, tile_size_type)

# 分散リンク
object_fifo_link("in", "in0", [0, 16, 32])
```

**出力Object FIFO:**

```python
# 結合リンク
object_fifo_link("out0", "out", [0, 16, 32])
```

### コア実行

各コンピュートタイルは、FIFOリストにインデックスを付けて適切な入力/出力チャネルにアクセスし、ループ構造内で同一のロジックを実行します。

```python
@core(ComputeTile1)
def core_body_1():
    for _ in range_(iterations):
        elem_in = of_in0_fifos[0].acquire(1)
        elem_out = of_out0_fifos[0].acquire(1)
        # 処理...
        of_in0_fifos[0].release(1)
        of_out0_fifos[0].release(1)
```

**重要な変更点:**
- 完全なデータセットではなく、タイルサイズのデータチャンクを処理
- 各コアは割り当てられたデータ部分のみを処理

## データ分散の詳細

### 分散（Distribute）パターン

```
全体データ: [0, 1, 2, ..., 47]  (48要素)

↓ 分散 (オフセット: [0, 16, 32])

Worker 1: [0, 1, ..., 15]   (16要素)
Worker 2: [16, 17, ..., 31] (16要素)
Worker 3: [32, 33, ..., 47] (16要素)
```

### 結合（Join）パターン

```
Worker 1 出力: [processed_0, ..., processed_15]
Worker 2 出力: [processed_16, ..., processed_31]
Worker 3 出力: [processed_32, ..., processed_47]

↓ 結合 (オフセット: [0, 16, 32])

全体出力: [processed_0, ..., processed_47]
```

## 並列実行の利点

### 性能向上

- **スループット**: 理想的には3倍の性能
- **レイテンシ**: データ分割により部分的に改善
- **リソース使用**: 複数のコアを効率的に活用

### スケーラビリティ

- コア数に応じて簡単にスケール
- 同じパターンで4コア、8コア以上に拡張可能
- メモリ階層を考慮した設計

## コンパイル

**高レベルIRON:**
```bash
make all
```

**明示的配置IRON:**
```bash
make placed
```

## 設計上の考慮事項

### ワークロード分散

- データを均等に分割
- 各Workerの計算量を同等に
- 負荷バランスの最適化

### 同期とタイミング

- すべてのWorkerが同時に開始
- データ依存関係の管理
- 結合時の順序保証

### メモリ管理

- メモリタイルの効率的な使用
- L1メモリの適切な割り当て
- バッファリング戦略の最適化

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-2/section-2e)を参照してください。

-----

[[前へ - Section 2d](../section-2d/README.md)] [[次へ - Section 2f](../section-2f/README.md)] [[トップ](../../README.md)]
