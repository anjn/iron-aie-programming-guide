# Object FIFO再利用パターン

前の[セクション](../../section-2a/index.html#object-fifoのオブジェクトへのアクセス)で、Object FIFOのacquireおよびrelease関数を組み合わせることで、データ再利用を伴うスライディングウィンドウの動作を実現できることが述べられました。具体的には、この通信パターンは、Object FIFOのプロデューサまたはコンシューマが、以前にacquireしたオブジェクトよりも少ない数のオブジェクトをreleaseする場合に発生します。Object FIFOからのacquireはデータを破壊しないため、releaseされていないオブジェクトはデータの新しいコピーを必要とせずに使用し続けることができます。

重要な点として、新しいacquire関数を呼び出すたびに、プロセスがアクセスできる新しいオブジェクトまたはオブジェクトの配列が返されますが、これには**以前のacquire呼び出しからのreleaseされていないオブジェクトが含まれます**。Object FIFOプリミティブを通じた適切な低レベル化を保証するために、プロセスは常に**最新の**acquire呼び出しの結果を使用してreleaseされていないオブジェクトにアクセスする必要があります。

以下の例では、`of0`は3つのオブジェクト（object0、object1、object2）の深さで作成されています。コンシューマWorkerで実行されるプロセスは次の図に示され、以下で詳しく説明されます。

```python
of0 = ObjectFifo(line_type, name="objfifo0", depth=3) # 3つのオブジェクト: object0, object1, object2

# 外部バイナリカーネル定義
test_fn2 = Kernel(
    "test_func2",
    "test_func2.cc.o",
    [line_type, line_type, np.int32],
)

# コアが実行するタスク
def core_fn(of_in, test_func2):
    ### 状況1
    elems = of_in.acquire(2) # object0とobject1をacquire
    test_func2(elems[0], elems[1], line_size)
    of_in.release(1) # object0をrelease

    ### 状況2
    elems_2 = of_in.acquire(2) # object2をacquire; object1は以前にacquireされていた
    test_func2(elems_2[0], elems_2[1], line_size)
    of_in.release(1) # object1をrelease

    ### 状況3
    elems_3 = of_in.acquire(2) # object0をacquire; object2は以前にacquireされていた
    test_func2(elems_3[0], elems_3[1], line_size)
    of_in.release(1) # object2をrelease

    ### 状況4
    elems_4 = of_in.acquire(2) # object1をacquire; object0は以前にacquireされていた

# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, [of0.cons(), test_fn2])
```

以下の図は、マークされた状況1から4のそれぞれにおけるシステムの状態を表しています。ここで、コンシューマWorkerはTile Bにマッピングされています（Tile Aは、上記のコードの一部ではない暗黙のプロデューサプロセスを実行しています）：

1. コンシューマBは最初に、変数`elems`で`of0`から2つの要素をacquireします。これがBが初めてacquireするため、object0とobject1にアクセスできます。次に、Bは2つのacquireされた要素に対して`test_func2`を適用します。最後に、Bは1つのオブジェクト（最も古くacquireされたもの）をreleaseし、object1を保持します。
2. Bは変数`elems_2`で2つの要素をacquireします。これで、object1（ステップ1の最初のacquire呼び出しからacquireされたまま）と、新しくacquireされたobject2にアクセスできます。Bは再び関数を適用し、その後1つのオブジェクトのみをreleaseし、object2を保持します。
3. Bは`elems_3`で2つのオブジェクトをacquireし、object2とobject0にアクセスできます。Bは1つのオブジェクトをreleaseし、object0を保持します。
4. Bは`elems_4`で2つのオブジェクトをacquireし、object0とobject1にアクセスできます。これにより、ステップ1の開始時の状況に戻ります。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/Reuse.png" height="400">

上記の状況は、4回の反復を持つ`for`ループに融合できます。コンシューマプロセスは、各反復でacquireした要素よりも1つ少ない要素を継続的にreleaseすることで、各反復で1つずつスライドする2つのオブジェクトのスライディングウィンドウの動作を実装しています：

```python
# ObjectFifosを使用したデータフロー
of0 = ObjectFifo(line_type, name="objfifo0", depth=3) # 3つのオブジェクト: object0, object1, object2

# 外部バイナリカーネル定義
test_fn2 = Kernel(
    "test_func2",
    "test_func2.cc.o",
    [line_type, line_type, np.int32],
)

# コアが実行するタスク
def core_fn(of_in, test_func2):
    for _ in range_(4):
        elems = of_in.acquire(2) # object0とobject1をacquire
        test_func2(elems[0], elems[1], line_size)
        of_in.release(1) # object0をrelease

# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, [of0.cons(), test_fn2])
```

---

**注意**: より詳細な情報については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-2/section-2b/01_Reuse)を参照してください。
