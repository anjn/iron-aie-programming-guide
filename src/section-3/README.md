# Section 3 - My First Program

<img align="right" width="500" height="250" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/binaryArtifacts.svg">

このセクションでは、AIE配列用の最初の完全なプログラムを作成します。ここでは、デバイスバイナリとホストバイナリの2つの異なるバイナリアーティファクトをコンパイルします。デバイスバイナリは、AIE配列の構成を含むXCLBINファイル（コンピュートコアのプログラムメモリ、データムーバーのバッファディスクリプタ、スイッチボックス設定など）と、外部メモリとの間のデータ移動を実行するシーケンス命令を含む`insts.bin`で構成されます。ホストバイナリは、デバイスバイナリをロードし、`insts.bin`シーケンスをトリガーし、何らかのテストベンチチェックを実行して結果を検証する実行可能プログラムです。

[セクション1](../section-1/index.html)と[セクション2](../section-2/index.html)で学んだ構造設計とデータ移動の概念を組み合わせて、ホストバイナリは主にC++またはPythonで作成され、[Xilinx RunTime (XRT)](https://github.com/Xilinx/XRT)および[AMD XDNA Driver](https://github.com/amd/xdna-driver)を使用してデバイスと通信します。[セクション4](../section-4/index.html)では、ホストコードを使用した性能測定とトレースについて詳しく説明します。

<img align="right" width="410" height="84" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/vectorScalarMul.svg">

このセクションで使用する例は、**ベクトルスカラー乗算**（`c = a * factor`）です。これは、[basic programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)ディレクトリにあります。このガイドセクションのコードスニペットの完全版は、[programming_examples](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/)ディレクトリで見つけることができます。入力ベクトル`a`は合計4096個のint32要素で構成され、1024個の要素を持つ4つのチャンクに分割されて処理されます。

## AIE配列の構造記述

<img align="right" width="150" height="400" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/vectorScalarMulPhysicalDataFlow.svg">

設計の記述は[aie2.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-3/aie2.py)ファイルにあります。その構造は[セクション1](../section-1/index.html)で紹介した`high-level IRON`スタイルを使用しており、`Worker`タスクを配置し、ホストからAIE配列へのシーケンスを定義します。この設計では、乗算を実行するコンピュートコアとともにshimDMAユニットを配置し、入力と出力データを外部メモリとの間で移動させます。簡単にするために、この[設計例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)では外部関数として定義されたバイナリカーネル（つまり、構造記述の外部でコンパイルされたカーネル）を使用します。

```python
tensor_size = 4096
tile_size = tensor_size // 4

# テンソル型の定義
tensor_ty = np.ndarray[(tensor_size,), np.dtype[np.int32]]
tile_ty = np.ndarray[(tile_size,), np.dtype[np.int32]]
scalar_ty = np.ndarray[(1,), np.dtype[np.int32]]

# 外部バイナリカーネルの定義
scale_fn = Kernel(
    "vector_scalar_mul_aie_scalar",
    "scale.o",
    [tile_ty, tile_ty, scalar_ty, np.int32],
)
```

### データ移動

<img align="right" width="300" height="300" src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/vector_scalar.svg">

次に、データ移動を設定する必要があります。これは[セクション2](../section-2/index.html)で詳しく説明されているObject FIFOを使用します。この例では、3つのObject FIFOがあります：2つの入力FIFO（入力ベクトル`a`用に1つとスカラー係数用に1つ）と1つの出力FIFO（出力ベクトル`c`用）です。各Object FIFOの深さは2です。これにより、ShimのDMAとCompute TileのDMAが並行して実行でき、一方がバッファへの書き込みを行っている間に、もう一方がバッファからの読み取りを行うことができます。

```python
# 入力データ移動
of_in = ObjectFifo(tile_ty, name="in")
of_factor = ObjectFifo(scalar_ty, name="infactor")

# 出力データ移動
of_out = ObjectFifo(tile_ty, name="out")
```

Object FIFOは、shimDMAと外部メモリ間のデータ転送を実行する際に使用されます。このデータ転送の実装は、ランタイムシーケンスで定義されます。ランタイムシーケンスについては、[セクション2d](../section-2/section-2d/index.html)で詳しく説明されています。現在の例では、`Runtime()`クラスを使用して、入力データ（`rt.fill()`）と出力データ（`rt.drain()`）のshimDMA操作を設定します。また、Workerタスクを開始します（`rt.start()`）。

```python
# AIE配列との間でデータを移動するランタイム操作
rt = Runtime()
with rt.sequence(tensor_ty, scalar_ty, tensor_ty) as (a_in, f_in, c_out):
    rt.start(my_worker)
    rt.fill(of_in.prod(), a_in)
    rt.fill(of_factor.prod(), f_in)
    rt.drain(of_out.cons(), c_out, wait=True)
```

### コンピュートコアのアクセスパターン

最後に、各Object FIFOのオブジェクトにアクセスするパターンを定義します。Object FIFOオブジェクトへのアクセスは、プロデューサとコンシューマのハンドルを介して実行されます。コンピュートコアは、Object FIFOオブジェクトの`acquire`（取得）と`release`（解放）を行います。

入力Object FIFO `of_in`と出力Object FIFO `of_out`については、各要素を取得し、外部カーネル関数`scale_fn`を介して処理し、それぞれのObject FIFOに解放します。これらの操作は、全4096要素を完全に処理するために、各反復で1024要素のチャンクを処理する4回のループで繰り返されます。スカラー係数Object FIFO `of_factor`については、コアがループを開始する前に要素を取得し、処理が完了した後に解放します。

```python
# コアが実行するタスク
def core_fn(of_in, of_factor, of_out, scale_scalar):
    elem_factor = of_factor.acquire(1)
    for _ in range_(4):
        elem_in = of_in.acquire(1)
        elem_out = of_out.acquire(1)
        scale_scalar(elem_in, elem_out, elem_factor, 1024)
        of_in.release(1)
        of_out.release(1)
    of_factor.release(1)


# タスクを実行するWorkerを作成
my_worker = Worker(core_fn, [of_in.cons(), of_factor.cons(), of_out.prod(), scale_fn])
```

## カーネルコード

カーネルコードは[vector_scalar_mul.cc](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-3/vector_scalar_mul.cc)ファイルにあります。この例では、汎用C++コードを使用したスカラープロセッサバージョンを使用します。関数シグネチャは次のとおりです：

```c
void vector_scalar_mul_aie_scalar(int32_t *a_in, int32_t *c_out,
                                  int32_t *factor, int32_t N) {
  for (int i = 0; i < N; i++) {
    c_out[i] = *factor * a_in[i];
  }
}
```

スカラー係数が配列ポインタとして渡されることに注意してください。これは、Object FIFO通信メカニズムが、スカラー係数をメモリ内の要素の配列として送信するためです。そのため、カーネルコード内で参照解除する必要があります。AIEコアのベクトルプロセッサ機能を活用する、より最適化されたベクトル化されたカーネル実装の例については、[セクション4](../section-4/index.html)を参照してください。

## ホストコード

### 概要

ホストコード設計は、C++（[test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-3/test.cpp)）またはPython（[test.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-3/test.py)）で記述できます。詳細については[セクション4b](../section-4/section-4b/index.html)を参照してください。一般的に、C++ホストコードは[C++ test utilities](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/runtime_lib/test_lib/test_utils.h)を、Pythonホストコードは[Python test utilities](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/utils/test.py)を利用して、コードを簡潔に保ちます。C++とPythonの両方のホストコードには、7つの主要なステップがあります。ここでは、C++バージョンに焦点を当てます。

### 1. プログラム引数の解析

C++ホストコードは、3つの必須引数を受け入れます：XCLBINファイルへのパス、カーネル名、シーケンス命令ファイルへのパス。オプションでverbosityフラグも受け入れます。この例では、`test_utils.h`を使用して引数を解析します。

```c
// プログラム引数の解析
po::options_description desc("Allowed options");
po::variables_map vm;
test_utils::add_default_options(desc);

test_utils::parse_options(argc, argv, desc, vm);
int verbosity = vm["verbosity"].as<int>();

// 設計定数の宣言
constexpr bool VERIFY = true;
constexpr int IN_SIZE = 4096;
constexpr int OUT_SIZE = IN_SIZE;
```

### 2. 命令シーケンスの読み込み

次に、シーケンス命令ファイルを読み込みます。このファイルには、外部メモリとの間のデータ移動を実行する命令が含まれています。

```c
// 命令シーケンスのロード
std::vector<uint32_t> instr_v =
    test_utils::load_instr_sequence(vm["instr"].as<std::string>());
```

### 3. XRT環境の作成

XRTランタイムを初期化し、デバイスとカーネルをロードします。

```c
xrt::device device;
xrt::kernel kernel;

test_utils::init_xrt_load_kernel(device, kernel, verbosity,
                                vm["xclbin"].as<std::string>(),
                                vm["kernel"].as<std::string>());
```

### 4. XRTバッファオブジェクトの作成

XRTは最大5つのinoutバッファをサポートし、3から始まる連続した`group_id`値を使用してマッピングされます。この例では、命令シーケンス用に1つ、入力ベクトル用に1つ、スカラー係数用に1つ、出力ベクトル用に1つの、合計4つのバッファオブジェクトを作成します。番号付けは、シーケンス定義の順序に従います。最初の引数が`group_id(3)`を受け取り、2番目が`group_id(4)`を受け取る、というように続きます。この番号付けの詳細については、[Python utils documentation](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/python/utils/README.md#configure-shimdma)を参照してください。

```c
// バッファオブジェクトの設定
auto bo_instr = xrt::bo(device, instr_v.size() * sizeof(int),
                        XCL_BO_FLAGS_CACHEABLE, kernel.group_id(1));
auto bo_inA = xrt::bo(device, IN_SIZE * sizeof(int32_t),
                        XRT_BO_FLAGS_HOST_ONLY, kernel.group_id(3));
auto bo_inFactor = xrt::bo(device, 1 * sizeof(int32_t),
                            XRT_BO_FLAGS_HOST_ONLY, kernel.group_id(4));
auto bo_outC = xrt::bo(device, OUT_SIZE * sizeof(int32_t),
                        XRT_BO_FLAGS_HOST_ONLY, kernel.group_id(5));
```

### 5. データの初期化と同期

バッファオブジェクトをデータで初期化し、ホストからデバイスのメモリに同期します。

```c
// 命令ストリームをxrtバッファオブジェクトにコピー
void *bufInstr = bo_instr.map<void *>();
memcpy(bufInstr, instr_v.data(), instr_v.size() * sizeof(int));

// バッファbo_inAを初期化
int32_t *bufInA = bo_inA.map<int32_t *>();
for (int i = 0; i < IN_SIZE; i++)
    bufInA[i] = i + 1;

// バッファbo_inFactorを初期化
int32_t *bufInFactor = bo_inFactor.map<int32_t *>();
int32_t scaleFactor = 3;
*bufInFactor = scaleFactor;

// バッファbo_outCをゼロクリア
int32_t *bufOut = bo_outC.map<int32_t *>();
memset(bufOut, 0, OUT_SIZE * sizeof(int32_t));

// ホストからデバイスのメモリに同期
bo_instr.sync(XCL_BO_SYNC_BO_TO_DEVICE);
bo_inA.sync(XCL_BO_SYNC_BO_TO_DEVICE);
bo_inFactor.sync(XCL_BO_SYNC_BO_TO_DEVICE);
bo_outC.sync(XCL_BO_SYNC_BO_TO_DEVICE);
```

### 6. AIEで実行して同期

カーネルを実行し、完了を待ち、結果をデバイスからホストのメモリに同期します。

```c
unsigned int opcode = 3;
auto run =
    kernel(opcode, bo_instr, instr_v.size(), bo_inA, bo_inFactor, bo_outC);
run.wait();

// デバイスからホストのメモリに同期
bo_outC.sync(XCL_BO_SYNC_BO_FROM_DEVICE);
```

### 7. テストベンチチェックの実行

最後に、出力をゴールデンリファレンスと比較して結果を検証します。

```c
// 出力をゴールデンと比較
int errors = 0;
if (verbosity >= 1) {
    std::cout << "Verifying results ..." << std::endl;
}
for (uint32_t i = 0; i < IN_SIZE; i++) {
    int32_t ref = bufInA[i] * scaleFactor;
    int32_t test = bufOut[i];
    if (test != ref) {
    if (verbosity >= 1)
        std::cout << "Error in output " << test << " != " << ref << std::endl;
    errors++;
    } else {
    if (verbosity >= 1)
        std::cout << "Correct output " << test << " == " << ref << std::endl;
    }
}
```

## プログラムの実行

設計をコンパイルして実行するには、次のコマンドを使用します：

```sh
make
```

```sh
make run
```

これにより、C++ホストコードを使用して設計がコンパイルおよび実行されます。Pythonテストベンチバリアントを実行するには、次のコマンドを使用します：

```sh
make
```

```sh
make run_py
```

C++とPythonの両方のテストベンチバリアントは、同じ`insts.bin`ファイルと同じXCLBINファイルを使用します。

## ホストコードテンプレートと設計プラクティス

すべてのホストコード設計は、同じ7つのステップに従います。設計がコンパイルされてハングしないようにするために、設計パラメータ（`IN_SIZE`、`OUT_SIZE`など）が、構造記述ファイル、カーネルソースファイル、ホストコードファイル全体で一貫していることを確認することが重要です。これらのパラメータは通常、Makefileで定義され、必要に応じて他のファイルに渡されます。これらのパラメータが一貫していない場合、システムがハングしたり、デバッグが非常に困難な動作が発生する可能性があります。詳細については、[セクション4b](../section-4/section-4b/index.html)を参照してください。この[設計例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul)を参照として使用して、独自の設計を作成できます。

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-3)を参照してください。
