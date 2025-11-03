# Section 1 - Basic AI Engine building blocks

AIE配列をプログラミングする際、その構造的な構成要素を宣言して設定する必要があります：ベクトル処理のための**コンピュートタイル**（compute tiles）、より大きなレベル2の共有スクラッチパッドとしての**メモリタイル**（memory tiles）、NPU外部メモリ（つまりメインメモリ）へのデータ移動をサポートする**シムタイル**（shim tiles）です。このプログラミングガイドでは、IRON Pythonライブラリを使用します。これにより、使用するAI Engineタイルの選択、各タイルが実行すべきコード、タイル間のデータ移動方法、CPU側からの設計呼び出し方法など、NPU設計全体を記述できます。後ほど、C/C++でのベクトルプログラミングを探求します。これは個々のコンピュートタイル用の計算カーネルを最適化するのに役立ちます。

## Pythonソースファイル（aie2.py）のウォークスルー

まず、IRON設計の最も高い抽象化レベルでの基本的なPythonソースファイル（[aie2.py](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_guide/section-1/aie2.py)という名前）を見てみましょう：

このPythonソースの先頭で、IRONライブラリを定義するモジュールをインポートします：高レベル抽象化構造のための`aie.iron`、リソース配置アルゴリズムのための`aie.iron.placers`、ターゲットアーキテクチャのための`aie.iron.device`です。

```python
from aie.iron import Program, Runtime, Worker, LocalBuffer
from aie.iron.placers import SequentialPlacer
from aie.iron.device import NPU1Col1, Tile
```

AIE配列内部のデータ移動も通常この段階で宣言されますが、設計設定のその部分は専用の[セクション](../section-2/index.html)があり、ここでは詳しく説明しません。

```python
# データフロー設定
# ガイドの今後のセクションで説明...
```

AIE配列では、計算カーネルはコンピュートタイル上で実行され、これは**Worker**で表されます。Workerは実行するルーチンと、それを実行するために必要な引数のリストを入力として受け取ります。Workerクラスは以下に定義されており、[worker.py](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/python/iron/worker.py)で見つけることができます。Workerは、AIE配列内の`placement`タイルに明示的に配置することも、このセクションでさらに説明するように、配置をコンパイラに任せることもできます。最後に、`while_true`入力はデフォルトでTrueに設定されています。これは、Workerが設計開始後に通常継続的に実行されるためです。

```python
class Worker(ObjectFifoEndpoint):
    def __init__(
        self,
        core_fn: Callable | None,
        fn_args: list = [],
        placement: PlacementTile | None = AnyComputeTile,
        while_true: bool = True,
    )
```

この単純な設計では、`core_fn`ルーチンを実行する1つのWorkerのみがあります。計算ルーチンはローカルデータバッファを繰り返し処理し、各エントリをゼロに初期化します。この場合、計算ルーチンには入力がありません。ガイドの次のセクションで見るように、計算タスクは通常、外部メモリからAIE配列に持ち込まれたデータ上で実行され、生成された出力は外部に送り返されます。この例では、WorkerはAIE配列の座標(0,2)のコンピュートタイルに明示的に配置されていることに注意してください。

```python
# Workerが実行するタスク
def core_fn():
    local = LocalBuffer(data_ty, name="local")
    for i in range_(data_size):
        local[i] = 0

# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, [], placement=Tile(0, 2), while_true=False)
```

> **注意 1:** `range_`のアンダースコアに気付きましたか？IRONはNPU設計を通常のPythonプログラムのように見せますが、ここで書くコードがNPU上で直接実行されるわけ _ではない_ ことを理解することが重要です。代わりに、IRON設計で書くコードは _他のコードを生成します_ （メタプログラミング）。これは、コード文字列を含むprint文を書くようなものです。その後、ツールチェーンがこの生成された他のコードをコンパイルし、それがNPU上で直接実行できるようになります。
>
> これは、上記の例で`range_`の代わりに`range`を書くと、生成されるNPUコードには多くの`local[i] = 0`命令が含まれますが、ループは全くないということを意味します（ループは「展開」され、バイナリが大きくなり、ループの反復回数はコード生成時に固定されている必要があります）。一方、`range_`を使用すると、Pythonはループ本体を1回だけ実行して（そこに含まれる命令を収集し）、NPUコードにループを発行します。そしてNPUがループを実行します。
> 同じことは`if`のような他の分岐構造にも当てはまります。Pythonのネイティブ構造を使用すると、NPUコードに実際の分岐が発行されないことを意味します！

> **注意 2:** 上記のコードのWorkerは`while_true=False`でインスタンス化されています。デフォルトでは、この属性は`True`に設定されており、その場合、タスクで表現されるカーネルコードは、ステップ1で`sys.maxsize`まで反復するforループでラップされます。これは、Workerのコードを無限にループさせる意図で`while(True)`をシミュレートします。一意の名前でローカルバッファを作成する場合など、タスクコードによっては、これによりコンパイラの問題が発生する可能性があります。

前のコードスニペットで、Worker間のデータ移動を設定する必要があると述べました。これには、`Runtime`シーケンス内で処理されるAIE配列との間のデータ移動は含まれません。プログラミングガイドには、ランタイムデータ移動の専用[セクション](../section-2/section-2d/index.html)があります。この例では、データ移動設定を詳しく見ないため、ランタイムシーケンスはWorkerを開始するだけです。

```python
# AIE配列との間でデータを移動するランタイム操作
rt = Runtime()
with rt.sequence(data_ty, data_ty, data_ty) as (_, _, _):
    rt.start(my_worker)
```

すべてのコンポーネントは、デバイス上で設計を実行するために必要なすべての設計情報を表す`Program`にまとめられます。また、この段階で、以前に配置されていないWorkerが`Placer`を使用してAIEタイルにマッピングされます。現在、IRONで利用可能な配置アルゴリズムは1つだけで、以下のコードスニペットに見られる`SequentialPlacer()`です。他のplacerは最小限の労力で追加でき、[placers.py](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/python/iron/placers.py)にあるこれらのツールを試すことをすべてのユーザーに推奨します。最後に、プログラムが印刷され、IRONライブラリとPython言語バインディングから対応するMLIR定義が生成されます。

```python
# デバイスタイプとランタイムからプログラムを作成
my_program = Program(NPU1Col1(), rt)

# コンポーネントを配置（デバイス上のリソースを割り当て）してMLIRモジュールを生成
module = my_program.resolve_program(SequentialPlacer())

# 生成されたMLIRを印刷
print(module)
```

> **注意:** 上記で説明または言及されたすべてのコンポーネントは、`resolvable`インターフェースを継承しており、`resolve()`関数が呼び出されるまでMLIR操作の作成を延期します。それが`Program`の`resolve_program()`関数のタスクであり、IRONクラスの1つがMLIR等価物を生成するのに十分な情報を持っていない場合、エラーが発生します。

## Pythonソースファイル（aie2_placed.py）のウォークスルー

IRONは、コンポーネントが座標を使用してAIEタイルに明示的に配置されるタイルレベルの粒度で設計を記述することもできます。このレベルでのIRON設計の基本的なPythonソースファイル（[aie2_placed.py](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_guide/section-1/aie2_placed.py)という名前）を再度見てみましょう。

このPythonソースの先頭で、IRON AIEライブラリ`aie.dialects.aie`とmlir-aieコンテキスト`aie.extras.context`を定義するモジュールをインポートします。これらはAI EnginesのMLIR定義にバインドされます。

```python
from aie.dialects.aie import * # プライマリmlir-aieダイアレクト定義
from aie.extras.context import mlir_mod_ctx # mlir-aieコンテキスト
```

次に、mlir-aieコンテキスト内から呼び出されたときにMLIRコードに展開される構造設計関数を宣言します（このサブセクションの最後の部分を参照）。

```python
# AI Engine構造設計関数
def mlir_aie_design():
    <... AI Engineデバイス、ブロック、接続 ...>
```

AI Engineデバイス、ブロック、接続の宣言方法を見てみましょう。まず、`@device(AIEDevice.npu1_1col)`または`@device(AIEDevice.npu2)`を介してAIEデバイスを宣言します。ブロックと接続自体は、`def device_body():`内で宣言されます。ここで、AI Engineブロックをインスタンス化します。この最初の例ではAIEコンピュートタイルです。

タイル宣言の引数はタイル座標（列、行）です。宣言された各タイルをPythonプログラムの変数に割り当てます。

> **注意:** プログラムが実行されるときにデバイス上で使用される実際のタイル座標は、ここで宣言されたものと異なる場合があります。たとえば、Ryzen™ AI上のNPU（`@device(AIEDevice.npu)`）では、これらの座標は相対座標である傾向があり、ランタイムスケジューラがランタイム中に別の利用可能な列に割り当てる場合があります。

```python
    # デバイス宣言 - ここではaie2デバイスNPUを使用
    @device(AIEDevice.npu1)
    def device_body():

        # タイル宣言
        ComputeTile1 = tile(1, 3)
        ComputeTile2 = tile(2, 3)
        ComputeTile3 = tile(2, 4)
```

コンピュートコアはコンピュートタイルにマッピングできます。また、コアの本体内から呼び出すことができる外部カーネル関数にリンクすることもできますが、それはこのセクションの範囲を超えており、ガイドでさらに説明されます。この例の設計では、コンピュートコアはローカルデータテンソルを宣言し、それを繰り返し処理し、各エントリをゼロに初期化します。

```python
        data_size = 48
        data_ty = np.ndarray[(data_size,), np.dtype[np.int32]]

        # コンピュートコア宣言
        @core(ComputeTile1)
        def core_body():
            local = buffer(ComputeTile1, data_ty, name="local")
            for i in range_(data_size):
                local[i] = 0
```

設計関数内でブロック（と接続）の宣言が完了したら、プログラムのメイン本体に移り、関数を呼び出してMLIRで設計を出力します。これは、まず`with mlir_mod_ctx() as ctx:`の行を介してMLIRコンテキストを宣言することによって行われます。これは、後続のインデントされたPythonコードがMLIRコンテキストにあることを示し、以前に定義した設計関数`mlir_aie_design()`を呼び出します。これは、設計関数内のすべてのコードがMLIRコンテキストにあると理解され、より詳細なMLIRブロック定義のIRONカスタムPythonバインディング定義を含むことを意味します。最後の行は`print(ctx.module)`で、MLIRコンテキストで定義されたコードを取得してstdoutに印刷します。これにより、Pythonバインドコードが対応するMLIRに変換され、stdoutに印刷されます。

```python
# 後続のコードがmlir-aieコンテキストにあることを宣言
with mlir_mod_ctx() as ctx:
    mlir_aie_design() # mlir-aieコンテキスト内で設計関数を呼び出す
    print(ctx.module) # PythonからMLIRへの変換をstdoutに印刷
```

## その他のタイルタイプ

コンピュートタイルの他に、AIE配列にはL3メモリにアクセスするためのデータムーバー（シムDMAとも呼ばれる）と、より大きなL2スクラッチパッド（メモリタイルと呼ばれる）も含まれます。これらはAIE-ML世代以降で利用可能です - [このプログラミングガイドの序文](../index.html)を参照してください。これらの他のタイプの構造ブロックの宣言は同じ構文に従いますが、特定のターゲットデバイスの物理レイアウトの詳細が必要です。シムDMAは通常行0を占め、メモリタイル（利用可能な場合）は行1に存在することが多いです。次のコードセグメントは、単一のNPU列で見つかるすべての異なるタイルタイプを宣言しています。

```python
    # デバイス宣言 - ここではaie2デバイスNPUを使用
    @device(AIEDevice.npu1)
    def device_body():

        # タイル宣言
        ShimTile     = tile(0, 0)
        MemTile      = tile(0, 1)
        ComputeTile1 = tile(0, 2)
        ComputeTile2 = tile(0, 3)
        ComputeTile3 = tile(0, 4)
        ComputeTile4 = tile(0, 5)
```

## 演習

1. コマンドラインからPythonプログラムを実行するには、`python3 aie2.py`と入力します。これにより、Python構造設計がMLIRソースコードに変換されます。設計環境にmlir-aie Pythonバインドダイアレクトモジュールが既に含まれている場合、コマンドラインから機能します。これを[Makefile](https://github.com/Xilinx/mlir-aie/blob/v1.1.1/programming_guide/section-1/Makefile)に含めたので、今すぐ`make`を実行してください。次に、`build/aie.mlir`で生成されたMLIRソースを確認します。

2. `make clean`を実行して生成されたファイルを削除します。Workerのコード（`core_fn`）で`range_`を`range`（アンダースコアなし）に置き換えます。何が起こると予想しますか？`build/aie.mlir`の生成されたコードを調査し、生成されたコードがどのように変更されたかを観察してください。
   <details>
   <summary>答えを見る</summary>
   生成されたMLIRコードにはループが含まれず、代わりに同じ命令が何度も繰り返されます。
   </details>

3. 再び`make clean`を実行します。次に、`sequence`を`sequenc`にスペルミスするなど、Pythonソースにエラーを導入し、再び`make`を実行します。どのようなメッセージが表示されますか？
   <details>
   <summary>答えを見る</summary>
   sequencが認識されないため、Pythonエラーがあります。
   </details>

4. 再び`make clean`を実行します。次に、`sequenc`を`sequence`に戻してエラーを変更しますが、Workerを座標(-1, 3)のタイルに配置します。これは無効な場所です。再び`make`を実行します。今度はどのようなメッセージが表示されますか？
   <details>
   <summary>答えを見る</summary>
   部分配置エラーがあります。
   </details>

5. 再び`make clean`を実行します。Workerタイルを元の座標に戻します。Workerから`while_true=False`属性を削除し、再び`make`を実行します。何が観察されますか？
   <details>
   <summary>答えを見る</summary>
   Workerタスクコードがforループ内にネストされています。
   </details>

6. 次に、配置されたバージョンのコードを見てみましょう。`make placed`を実行し、`build/aie_placed.mlir`で生成されたMLIRソースを確認します。

7. `make clean`を実行して生成されたファイルを削除します。`ComputeTile1`の座標を(-1,3)に変更して、上記と同じエラーを導入します。再び`make placed`を実行します。今度はどのようなメッセージが表示されますか？
   <details>
   <summary>答えを見る</summary>
   エラーは生成されません。
   </details>

8. エラーは生成されませんが、コードは無効です。`build/aie_placed.mlir`で生成されたMLIRコードを確認してください。この生成された出力は無効なMLIR構文であり、このMLIRソースでmlir-aieツールを実行するとエラーが生成されます。ただし、関数`ctx.module.operation.verify()`を使用すると有効化できる追加のPython構造構文チェックがあります。これは、Pythonバインドコードがmlir-aieコンテキスト内で有効な操作を持っているかどうかを検証します。

    次のようなコードブロックを使用して、`ctx.module.operation.verify()`のチェックで`print(ctx.module)`呼び出しを限定します：
    ```python
    res = ctx.module.operation.verify()
    if res == True:
        print(ctx.module)
    else:
        print(res)
    ```
    この変更を行い、再び`make placed`を実行します。今度はどのようなメッセージが表示されますか？
    <details>
    <summary>答えを見る</summary>
    最小値が0であるため、'column value fails to satisfy the constraint'と表示されるようになります。
    </details>

---

**注意**: このページは[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-1)の非公式日本語訳です。
