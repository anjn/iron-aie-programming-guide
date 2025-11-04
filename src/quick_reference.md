# <ins>IRONクイックリファレンス</ins>

* [Pythonバインディング](#pythonバインディング)
* [Pythonヘルパー関数](#pythonヘルパー関数)
* [カーネルプログラミングのための一般的なAIE API関数](#カーネルプログラミングのための一般的なaie-api関数)
* [役立つAI Engineアーキテクチャリファレンスとテーブル](#役立つai-engineアーキテクチャリファレンスとテーブル)
* [AI Engineドキュメント](#ai-engineドキュメント)
* [AIE詳細リファレンス](#aie詳細リファレンス)

----

## Pythonバインディング

| 関数シグネチャ  | 定義 | パラメータ | 戻り値の型 | 例 |
|---------------------|------------|------------|-------------|---------|
| `tile(column, row)` | AI Engineタイルを宣言 | `column`: 列インデックス番号 <br> `row`: 行インデックス番号 | `<tile>` | ComputeTile = tile(1,3) |
| `external_func(name, inputs, output)` | AIEコア上で実行される外部カーネル関数を宣言|  `name`: 外部関数名 <br> `input`: 入力型のリスト <br> `output`: 出力型のリスト | `<external_func>` | scale_scalar = external_func("vector_scalar_mul_aie_scalar", inputs=[tensor_ty, tensor_ty, np.ndarray[(1,), np.dtype[np.int32]]]) | |
| `npu_dma_memcpy_nd(metadata, bd_id, mem, sizes)` | 外部メモリにアクセスするn次元DMAを設定 | `metadata`:  ObjectFifo pythonオブジェクトまたは`object_fifo`の名前文字列<br> `bd_id`: 識別番号<br> `mem`: 転送用メモリ<br> `sizes`: 4Bの粒度での4次元転送サイズ | `None` | npu_dma_memcpy_nd(metadata="out", bd_id=0, mem=C, sizes=[1, 1, 1, N]) |
| `dma_wait(object_fifo, ...)` | 外部メモリにアクセスするためのホスト-ShimDMA同期を設定 | `metadata`: 完了を待機しているObjectFifo（Pythonオブジェクトまたは名前文字列）を識別します。これは可変引数関数であり、1つ以上のメタデータを一度に受け取り、与えられた順序で待機します | `None` | dma_wait(of_out) |
| `npu_sync(column, row, direction, channel, column_num=1, row_num=1)` | 外部メモリにアクセスするためのホスト-ShimDMA同期を設定する代替方法 | `column`と`row`: 同期を開始するタイル位置を指定 <br> `direction`: DMAの方向を示します（0はホストへの書き込み、1はホストからの読み取り） <br> `channel`: 同期トークンのDMAチャネル（0または1）を識別 <br> `column_num`と`row_num`（オプション）: 同期を待機するタイルの範囲を定義| `None` | npu_sync(column=0, row=0, direction=0, channel=1) |
| **Object FIFO** |||
| `object_fifo(name, producerTile, consumerTiles, depth, datatype)` | Object FIFOを構築 | `name`: Object FIFO名 <br> `producerTile`: プロデューサタイルオブジェクト <br> `ConsumerTiles`: コンシューマタイルオブジェクトのリスト <br> `depth`: Object FIFO内のオブジェクト数 <br> `datatype`: Object FIFO内のオブジェクトの型| `<object_fifo>` | of0 = object_fifo("objfifo0", A, B, 3, np.ndarray[(256,), np.dtype[np.int32]]) |
| `<object_fifo>.acquire(port, num_elem)` | Object FIFOからacquire | `port`: `ObjectFifoPort.Produce`または`ObjectFifoPort.Consume` <br> `num_elem`: acquireするオブジェクト数 | `<objects>` | elem0 = of0.acquire(ObjectFifoPort.Produce, 1) |  |
| `object_fifo.release(port, num_elem)` | Object FIFOからrelease | `port`: `ObjectFifoPort.Produce`または`ObjectFifoPort.Consume` <br> `num_elem`: | `None` | of0.release(ObjectFifoPort.Consume, 2) |
| `object_fifo_link(fifoIns, fifoOuts)` | Object FIFO間のリンクを作成 | `fifoIns`: Object FIFOのリスト（変数または名前）<br> `fifoOuts`: Object FIFOのリスト（変数または名前） | `None` | object_fifo_link(of0, of1) |
| **ルーティングバインディング（トレースと低レベル設計に関連）** |||
| `flow(source, source_bundle, source_channel, dest, dest_bundle, dest_channel)` | 送信元と宛先の間に回路交換フローを作成 | `source`: フローの送信元タイル <br> `source_bundle`: 送信元WireBundleの型（完全なリストは[AIEAttrs.td](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/include/aie/Dialect/AIE/IR/AIEAttrs.td)を参照） <br> `source_channel`: 送信元チャネルインデックス <br> `dest`: フローの宛先タイル <br> `dest_bundle`: 宛先WireBundleの型（完全なリストは[AIEAttrs.td](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/include/aie/Dialect/AIE/IR/AIEAttrs.td)を参照） <br> `dest_channel`: 宛先チャネルインデックス | `None` | flow(ComputeTile, WireBundle.DMA, 0, ShimTile, WireBundle.DMA, 1) | トレース用のルーティングの場合、srcPortとsrcChannelはそれぞれWireBundle.Traceと0になります|
| `packetflow(pkt_id, source, source_port, source_channel, dest, dest_port, dest_channel, keep_pkt_header)` | 送信元と宛先の間にパケット交換フローを作成 | `pkt_id`: 一意のパケットID <br>  `source`: パケットフローの送信元タイル <br> `source_port`: 送信元WireBundleの型（完全なリストは[AIEAttrs.td](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/include/aie/Dialect/AIE/IR/AIEAttrs.td)を参照） <br> `source_channel`: 送信元チャネルインデックス <br> `dest`: パケットフローの宛先タイル <br> `dest_port`: 宛先WireBundleの型（完全なリストは[AIEAttrs.td](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/include/aie/Dialect/AIE/IR/AIEAttrs.td)を参照） <br> `dest_channel`: 宛先チャネルインデックス <br>`keep_pkt_header`: ヘッダーを保持するブールフラグ | `None` | packetflow(1, ComputeTile2, WireBundle.Trace, 0, ShimTile, WireBundle.DMA, 1, keep_pkt_header=True) | 例はトレースルーティングを示しています。コアメモリトレースユニットからルーティングする場合は、チャネル1を使用します |
|||||

> **注意：** `tile`: デバイス上で実際に実行されるタイル座標は、ここで宣言されたものと異なる場合があります。例えば、Ryzen AIでは、これらの座標は相対座標である傾向があり、ランタイムスケジューラが利用可能な別の列に割り当てる可能性があります。

> **注意：** `object_fifo`: `producerTile`と`consumerTiles`の入力はAI Engineタイルです。`consumerTiles`は、複数のコンシューマの場合、タイルの配列として指定することもできます。

> **注意：** `<object_fifo>.{acquire,release}`: 出力は単一のオブジェクトまたはオブジェクトの配列のいずれかであり、配列のようにインデックスを付けることができます。

> **注意：** `object_fifo_link` リンクで共有タイルとして使用されるタイルは、現在Memタイルである必要があります。入力`fifoIns`と`fifoOuts`は、単一のObject FIFOまたはそれらのリストのいずれかです。両方とも、python変数または名前のいずれかを使用して指定できます。現在、2つの入力のいずれかがObjectFIFOのリストである場合、もう一方は単一のObject FIFOのみになります。

## Pythonヘルパー関数
| 関数シグネチャ | 説明 |
|--------------------|-------------|
| `print(ctx.module)` | ctxでラップされた構造コードをmlirに変換し、標準出力に出力します|
| `ctx.module.operation.verify()` | Pythonバインディングされたソースコードに対して追加の構造検証を実行し、結果を標準出力に返します |

## カーネルプログラミングのための一般的なAIE API関数
| 関数シグネチャ  | 定義 | パラメータ | 戻り値の型 | 例 |
|---------------------|------------|------------|-------------|---------|
| `aie::vector<T, vec_factor> my_vector` | ベクトル型を宣言 | `T`: データ型 <br> `vec_factor`: ベクトル幅 | n/a | aie::vector<int16_t, 32> my_vector; |
| `aie::load_v<vec_factor>(pA1);` | ベクトルロード | `vec_factor`: ベクトル幅 | `aie::vector` | aie::vector<int16_t, 32> my_vector; |

## 役立つAI Engineアーキテクチャリファレンスとテーブル
* [AIE2 - サポートされているデータ型とベクトルサイズのテーブル（AIE API）](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/group__group__basic__types.html)

* 役立つタイルコアトレースイベント
    | 一般的なイベント | イベントID | 10進数値 |
    |--------------------|----------|-----------|
    | True                       |0x01| 1 |
    | ストリームストール              |0x18| 24 |
    | コア命令 - イベント0  |0x21| 33|
    | コア命令 - イベント1  |0x22| 34 |
    | ベクトル命令（例：VMAC、VADD、VCMP） |0x25|  37 |
    | ロックacquireリクエスト      |0x2C|  44 |
    | ロックreleaseリクエスト      |0x2D|  45 |
    | ロックストール                 |0x1A|  26 |
    | コアポート実行中 1        |0x4F|  79 |
    | コアポート実行中 0        |0x4B|  75 |
    * コアタイル、コアメモリ、メムタイル、shimタイルのイベントのより包括的なリストは、[このヘッダーファイル](https://github.com/Xilinx/aie-rt/blob/main-aie/driver/src/events/xaie_events_aie.h)にあります

## AI Engineドキュメント
* [UG1076のドキュメントリンク概要](https://docs.amd.com/r/en-US/ug1076-ai-engine-environment/Documentation)
* [AIE1アーキテクチャマニュアル - AM009](https://docs.amd.com/r/en-US/am009-versal-ai-engine/Overview)
* [AIE1レジスタリファレンス - AM015](https://docs.amd.com/r/en-US/am015-versal-aie-register-reference/Overview)
* [AIE2アーキテクチャマニュアル - AM020](https://docs.amd.com/r/en-US/am020-versal-aie-ml/Overview)
* [AIE2レジスタリファレンス - AM025](https://docs.amd.com/r/en-US/am025-versal-aie-ml-register-reference/Overview)
* [AIE APIユーザーガイド - v2023.2](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/index.html)
* [AIE1イントリンシクスユーザーガイド - v2023.2](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_intrinsics/intrinsics/index.html)
* [AIE2イントリンシクスユーザーガイド - v2023.2](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_ml_intrinsics/intrinsics/index.html)

## AIE詳細リファレンス
* [AIE2 - サポートされているデータ型とベクトルサイズのテーブル（AIE API）](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/group__group__basic__types.html)
