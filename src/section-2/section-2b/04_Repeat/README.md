# Object FIFO繰り返しパターン

低レベルのObject FIFOは、プロデューサからのデータを繰り返すことを指定する方法をユーザーに提供します。この機能は、次の構文を使用して利用できます：

```python
of0 = object_fifo("objfifo0", A, B, 2, np.ndarray[(256,), np.dtype[np.int32]])
of0.set_repeat_count(2) # 各オブジェクトのデータはコンシューマCに2回送信されます
```

この繰り返しは、Object FIFOのプロデューサタイルのダイレクトメモリアクセス（DMA）を使用して実現されます。特に、DMAバッファディスクリプタは、データの破損を避けるために、データが正しいタイミングで処理されることを保証する同期ロジックに依存しています。繰り返しパターンをプログラムするために、プロデューサタイルのバッファディスクリプタに関連する同期ロジックは、データの追加コピーを送信するように生成されます。これらのデータコピーは、以下の図の赤い矢印で示されているように、DMAレベルで作成されるため、追加のメモリが割り当てられることはありません：

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/RepeatSharedTile.png" height="300">

DMAとそのバッファディスクリプタの詳細については、[セクション2aの高度なトピック](../../section-2a/index.html#高度なトピックデータ移動アクセラレータ)および[セクション2g](../../section-2g/index.html)を参照してください。

繰り返しパターンは同期ロジックに依存しているため、Object FIFOの低レベル化では、利用可能な情報を使用してObject FIFOの`acquire`および`release`操作の値を変更し、DMAが繰り返すことを可能にするために計算タイルによって十分なトークンが生成され、これらのトークンがDMA繰り返し後の最初の`acquire`操作によって考慮されることを保証します。深さが1より大きいObject FIFOに対してこの調整を行うことは自明ではなく、現在サポートされていません。

この機能の特殊性の1つは、サイズが1より大きいObject FIFOの繰り返しパターンです。Object FIFOに対して生成されるデータ移動は、First In First Outの循環パターンに従い、これが繰り返しと組み合わされると、個々のオブジェクトの繰り返しではなく、循環パターン全体の繰り返しになります。これは、以下の図に示されており、赤い矢印は繰り返し値を表しています：

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/RepeatSharedTile_2.png" height="300">

具体的には、上の図で見られるパターンは次のようになります：`buff_ping - buff_pong - buff_ping - buff_pong`。ここで、各バッファのデータは各インスタンスで同じままです。

## リンクと繰り返し

前のセクションで説明したリンクで繰り返しを使用することもできます。次の構文を使用します：

```python
of0 = object_fifo("objfifo0", A, B, 2, np.ndarray[(256,), np.dtype[np.int32]])
of1 = object_fifo("objfifo1", B, C, 2, np.ndarray[(256,), np.dtype[np.int32]])
object_fifo_link(of0, of1)
of1.set_repeat_count(2) # 各オブジェクトのデータはコンシューマCに2回送信されます
```

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/Repeat.png" height="150">

この場合、繰り返しはObject FIFOリンクの共有タイルのダイレクトメモリアクセス（DMA）を使用して実現されます。

特に、繰り返し機能は前のセクションで紹介されたdistributeパターンと組み合わせて使用できます。現在、機能的正しさを保証するために、各distribute宛先に指定された繰り返し値は同じである必要があります。さらに、現在の構文は、同じdistributeパターン内で繰り返しありと繰り返しなしの両方の出力Object FIFOを同時にサポートしていません。以下のコードは、distributeパターンの2つの出力Object FIFOをそれぞれ3回繰り返すように設定する方法を示しています：

```python
of0 = object_fifo("objfifo0", A, B, 2, np.ndarray[(256,), np.dtype[np.int32]])
of1 = object_fifo("objfifo1", B, C, 2, np.ndarray[(256,), np.dtype[np.int32]])
of2 = object_fifo("objfifo2", B, D, 2, np.ndarray[(256,), np.dtype[np.int32]])
object_fifo_link(of0, [of1, of2])
of1.set_repeat_count(3)
of2.set_repeat_count(3)
```

上記のコードスニペットは、[こちら](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/test/npu-xrt/objectfifo_repeat/distribute_repeat/)にあるテストの一部です。

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2b/04_Repeat)を参照してください。
