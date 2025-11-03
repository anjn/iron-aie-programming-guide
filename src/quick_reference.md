# クイックリファレンス

このドキュメントは、AMDのAI Engineプログラミング用Pythonベースインターフェースである**IRON**のクイックリファレンスを提供します。

## Pythonバインディング

### コアタイル操作

**タイル宣言:**
```python
tile(column, row)
```
座標によってAI Engineタイルを宣言します。ただし、Ryzen AIなどのランタイム環境では、実際のデバイス座標が異なる場合があります。

**外部関数宣言:**
```python
external_func(name, inputs, output)
```
AIEコア上で実行されるカーネル関数を宣言します。関数名と型指定された入力/出力仕様が必要です。

### メモリとDMA設定

**n次元DMA転送:**
```python
npu_dma_memcpy_nd()
```
4Bの粒度を使用して、外部メモリへのn次元DMA転送を設定します。

**DMA同期:**
```python
dma_wait()
```
ホスト-ShimDMAアクセスを同期します。1つまたは複数のObjectFifo参照を受け入れます。

**NPU同期:**
```python
npu_sync()
```
方向パラメータ（0=ホストへの書き込み、1=ホストからの読み取り）とチャネル選択を使用した代替同期を提供します。

### Object FIFO操作

**Object FIFO構築:**
```python
object_fifo()
```
指定された深さとデータ型を持つ、プロデューサタイルとコンシューマタイル間の通信チャネルを構築します。

**オブジェクトアクセス:**
```python
.acquire()  # プロデューサまたはコンシューマポートからオブジェクトを取得
.release()  # オブジェクトを解放
```

**FIFO接続:**
```python
object_fifo_link()
```
共有メモリタイルを介して複数のFIFOを接続します。

### ルーティング（トレースと低レベル設計）

**回路交換接続:**
```python
flow()
```
WireBundleタイプとチャネルを指定して、タイル間の回路交換接続を作成します。

**パケット交換ルート:**
```python
packetflow()
```
一意のIDとオプションのヘッダー保存を使用して、パケット交換ルートを確立します。

## ヘルパー関数

**MLIR変換:**
```python
print(ctx.module)
```
PythonコードをMLIR形式に変換します。

**構造検証:**
```python
ctx.module.operation.verify()
```
構造的検証を実行します。

## カーネルプログラミングAPI

**ベクトル操作:**
```python
aie::vector<T, vec_factor>  # ベクトル宣言
aie::load_v<vec_factor>()   # 設定可能な幅とデータ型でのロード
```

## アーキテクチャリファレンス

### トレースイベントID

コアアクティビティをドキュメント化するイベントID：

- **0x18**: ストリームストール
- **0x25**: ベクトル命令
- **0x2C-0x2D**: ロック操作
- **0x4B, 0x4F**: コアポート実行状態

## データ型

**整数型:**
- `i8`, `i16`, `i32`, `i64`

**浮動小数点型:**
- `f32`: 32ビット浮動小数点
- `bf16`: BFloat16

**ベクトル型:**
```python
vector<N x type>  # N個の要素を持つベクトル
```

例：`vector<16 x i32>`、`vector<32 x bf16>`

## よく使うパターン

### 基本的なデータフロー設定

1. **デバイスとタイルの宣言**
2. **Object FIFOの作成**
3. **コンピュートコアの定義**
4. **ランタイムシーケンスの設定**

### Object FIFOの典型的な使用法

```python
# Object FIFOの作成
of_in = object_fifo("input", ShimTile, ComputeTile, 2, data_type)

# コンピュートコアでの使用
@core(ComputeTile)
def core_body():
    for _ in range_(iterations):
        elem_in = of_in.acquire(1)  # 1要素取得
        # 処理...
        of_in.release(1)            # 解放
```

## ドキュメントリソース

### 公式リファレンス

- [AIE1 Architecture Manual - AM009](https://docs.amd.com/r/en-US/am009-versal-ai-engine)
- [AIE2 Architecture Manual - AM020](https://docs.amd.com/r/en-US/am020-versal-aie-ml)
- [MLIR-AIE公式リポジトリ](https://github.com/Xilinx/mlir-aie)

### API詳細

より詳細なAPI情報とイントリンシクスドキュメントについては、Xilinx/AMDの公式ドキュメントを参照してください。

---

**注意**: このクイックリファレンスは概要を提供します。完全な詳細については、[公式quick_reference.md](https://github.com/Xilinx/mlir-aie/blob/main/programming_guide/quick_reference.md)を参照してください。

[[トップに戻る](./README.md)]
