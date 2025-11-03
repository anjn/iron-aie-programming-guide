# Section 4b - トレース

## 目的とコンテキスト

このドキュメントは、性能分析のためのAIE（AI Engine）トレース機能について説明します。「AIEハードウェアを最大限に最適化したいカーネルプログラマにとって、AIEコアとデータムーバーがどれだけ効率的に実行されているかを確認できることは重要です。」AIE2トレースユニットは、サイクル精度のハードウェアイベント可視性を提供します。

## 4ステップ実装プロセス

### ステップ1: トレース設定の有効化

ユーザーは、ランタイム関数を通じてトレースユニットを設定します。2つのアプローチがあります：

**方法A - ランタイム指定:**

```python
rt.enable_trace(trace_size, workers=[my_worker])
```

**方法B - Worker宣言:**

```python
worker = Worker(
    core_body,
    fn_args=[...],
    trace=1,
)
rt.enable_trace(trace_size)
```

**重要:** 「ランタイムシーケンスの`enable_trace`の`workers`引数は、常にワーカーの`trace=1`引数よりも優先されます。」

#### カスタマイズオプション

高度な設定パラメータには以下が含まれます：

- **`trace_offset`**: バッファオフセット（バイト単位、デフォルト0）
- **`ddr_id`**: ターゲットXRTバッファ選択
- **`coretile_events`**: コアタイルで監視する8つのイベント
- **`memtile_events`**: メモリタイルで監視する8つのイベント
- **`shimtile_events`**: シムタイルで監視する8つのイベント

### ステップ2: ホストコード設定

#### XRTバッファマッピング

標準パターンでのトレースサポートの例マッピング：

| inout0 (3) | inout1 (4) | inout2 (5) | inout3 (6) | inout4 (7) |
|------------|------------|------------|------------|------------|
| Input A    | Output C   | —          | —          | Trace      |

#### C/C++実装

```cpp
// トレースバッファの作成
auto bo_trace = xrt::bo(
    device,
    tmp_trace_size,
    XRT_BO_FLAGS_HOST_ONLY,
    kernel.group_id(7)  // group_id 7をトレースに使用
);

// バッファのマッピングと初期化
char *bufTrace = bo_trace.map<char *>();
memset(bufTrace, 0, myargs.trace_size);

// デバイスに同期
bo_trace.sync(XCL_BO_SYNC_BO_TO_DEVICE);

// カーネル実行...

// デバイスから同期
bo_trace.sync(XCL_BO_SYNC_BO_FROM_DEVICE);

// トレースデータを書き出し
test_utils::write_out_trace(
    (char *)bufTrace,
    myargs.trace_size,
    myargs.trace_file
);
```

#### Python実装

```python
# トレースバッファの登録
app.register_buffer(
    7,
    shape=trace_buf_shape,
    dtype=trace_buf_dtype
)

# 実行
full_output, trace_buffer = execute(
    app,
    in1_data,
    in2_data,
    enable_trace,
    trace_after_output
)

# トレースデータの処理
trace_buffer = trace_buffer.view(np.uint32)
write_out_trace(trace_buffer, str(opts.trace_file))
```

### ステップ3: トレースデータの解析

コマンドライン実行：

```bash
python parse_trace.py \
    --input trace.txt \
    --mlir build/aie_trace.mlir \
    --output trace_4b.json
```

これにより、可視化ツールと互換性のあるJSON波形ファイルが生成されます。

**入力ファイル:**
- `trace.txt`: 生のトレースデータ
- `build/aie_trace.mlir`: AIE設計のMLIR表現

**出力:**
- `trace_4b.json`: Perfetto形式の波形データ

### ステップ4: 可視化

生成されたJSONファイルを https://ui.perfetto.dev で開きます。

**ナビゲーション:**
- **A/D キー**: パン（左右移動）
- **W/S キー**: ズーム（拡大/縮小）
- **マウス**: クリックでイベントの詳細を表示

**表示される情報:**
- タイムライン上のイベント
- イベントの期間
- 各タイルのアクティビティ
- ロック操作とストール

## 一般的なトレースイベント

ドキュメントでは、標準的に監視されるイベントを特定しています：

### カーネル境界マーカー

- **`INSTR_EVENT_0`**: カーネル開始マーカー
- **`INSTR_EVENT_1`**: カーネル終了マーカー

### 実行イベント

- **`INSTR_VECTOR`**: ベクトル演算実行

### ポートアクティビティ

- **`PORT_RUNNING_0` ~ `PORT_RUNNING_7`**: ポートアクティビティ監視

### ロックイベント

- **`LOCK_STALL`**: ロック競合検出
- **`INSTR_LOCK_ACQUIRE_REQ`**: ロック取得リクエスト追跡
- **`INSTR_LOCK_RELEASE_REQ`**: ロック解放リクエスト追跡

## デバッグガイダンス

### よくある問題と解決策

#### 1. 空のトレースファイル

**原因:**
- 誤ったXRTバッファ選択
- group_idの不一致

**解決策:**
```cpp
// AIE設計とホストコード間でgroup_idが一致していることを確認
kernel.group_id(7)  // トレース用
```

#### 2. データ不足

**原因:**
- コア設計が最小限のイベントしか生成しない

**解決策:**
- ShimTileを追加
- DMAバースト長を64Bに削減してより頻繁なイベントを生成

```python
# DMA設定の調整
dma_burst_length = 64  # 小さいバーストでイベント増加
```

#### 3. オフセットエラー

**原因:**
- バッファ共有時のオフセット不一致

**解決策:**
```cpp
// トレースオフセットが出力バッファサイズと正確に一致することを確認
trace_offset = output_buffer_size;
```

#### 4. タイルルーティング

**原因:**
- マルチコア設計での誤ったタイル-シムDMA接続

**解決策:**
```python
# 正しいタイル接続を確認
flow(compute_tile, WireBundle.DMA, 0,
     shim_tile, WireBundle.DMA, 1)
```

#### 5. カラムシフト

**デバイスごとの要件:**
- **Phoenixデバイス**: `colshift=1`が必要
- **Strixデバイス**: `colshift=0`が必要

```python
# parse_trace.py実行時に指定
python parse_trace.py --colshift 1  # Phoenix
python parse_trace.py --colshift 0  # Strix
```

#### 6. バッファ割り当て

**実験的回避策:**

間欠的な失敗を避けるために、XRTバッファをトレースサイズの4倍で割り当てることを推奨：

```cpp
// 推奨サイズ
auto bo_trace = xrt::bo(
    device,
    trace_size * 4,  // 4倍のサイズ
    XRT_BO_FLAGS_HOST_ONLY,
    kernel.group_id(7)
);
```

## トレース分析のワークフロー

### 1. 設計とコンパイル

```bash
make
```

### 2. トレース有効で実行

```bash
make run
# または特定のトレース設定で
./test.exe --trace_size 8192 --trace_file trace.txt
```

### 3. トレースデータを解析

```bash
python parse_trace.py \
    --input trace.txt \
    --mlir build/aie_trace.mlir \
    --colshift 1 \
    --output trace.json
```

### 4. 可視化と分析

```bash
# ブラウザでhttps://ui.perfetto.devを開く
# trace.jsonをアップロード
```

### 5. 最適化

トレースから得た洞察に基づいて設計を改善：

- ストールの特定と削減
- データフローの最適化
- ロック競合の解消
- パイプライン効率の改善

## テンプレートカスタマイズ

実装のためのカスタマイズ可能な要素：

### Makefileでの設定

```makefile
# 入力/出力バッファサイズ
BUFFER_SIZE = 4096

# トレースサイズ
TRACE_SIZE = 8192

# データ型
DTYPE = int32_t
```

### データ型定義

PythonとC/C++間で一致させる：

```python
# Python
data_type = np.int32

# C/C++
using data_t = int32_t;
```

### バッファ初期化

```cpp
// カスタム初期化関数
void init_buffer(data_t* buffer, size_t size) {
    for (size_t i = 0; i < size; i++) {
        buffer[i] = /* 初期化ロジック */;
    }
}
```

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide/section-4/section-4b)を参照してください。

-----

[[前へ - Section 4a](../section-4a/README.md)] [[次へ - Section 4c](../section-4c/README.md)] [[トップ](../../README.md)]
