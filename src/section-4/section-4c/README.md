# Section 4c - カーネルのベクトル化と最適化

アプリケーション全体の実行時間を測定する方法（[セクション4a](../section-4a/index.html)）を学び、トレースを使用してカーネル性能を確認する方法（[セクション4b](../section-4b/index.html)）を見てきました。次は、カーネルのベクトル化をより詳しく見て、トレースを使用して性能を比較します。カーネルのベクトル化の概念を説明するために、以前のセクション4の例のローカルコピーではなく、[ベクトルスカラー乗算の例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)に切り替えます。なお、このデザイン例はデフォルトで16ビットデータを使用しており（以前のセクション4の例では32ビット）、`vectorized=True`に設定されています。

まず[ベクトルスカラー乗算](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)のデザイン例の概要を読んで、この例のさまざまなコンポーネントのアイデアを把握してください。次に、カーネルソースファイル（[scale.cc](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/aie_kernels/aie2/scale.cc)）をより詳しく見てみましょう。

[scale.cc](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/aie_kernels/aie2/scale.cc)では、スカラーコードが比較的単純で、[セクション4b](../section-4b/index.html)で使用したスカラーコードに似ていることがわかります：

```cpp
template <typename T>
void scale_scalar(T *a, T *c, T factor, const int32_t N) {
  event0();
  for (int i = 0; i < N; i++) {
    c[i] = factor * a[i];
  }
  event1();
}
```

ここでは、コードが入力ベクトル（`a`）を反復処理し、ベクトルの各要素をスカラー値（`factor`）で乗算してから、結果を出力ベクトル（`c`）に格納します。このシンプルなC/C++コードは、forループと、ループ内の単純な読み取りとスカラー乗算演算で構成されています。

## AIE API

これをベクトル化するには、まずAIE APIに慣れる必要があります。AIE APIは、基礎となるAIEプロセッサと関連する低レベルイントリンシクスを、より高レベルのC++ APIで抽象化します。AIE API（2023.2 Vitisツール）のドキュメントは[こちら](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/group__group__arithmetic.html#gafdca71673bdae6c6642b88dab9aee1fe)にあります。ベクトル×スカラー乗算の詳細を表示するには、左側のペインで*AI Engine API User Guide -> API Reference -> Arithmetic*に移動し、最初の`aie::mul`を選択します。これは`Vec * E`を示しており、`E`はスカラーintのような基本データ型です。

カーネルコードでこのAIE API関数を使用できるようにするには、まずカーネルソースにAIE APIヘッダーをインクルードする必要があります。

```cpp
#include <aie_api/aie.hpp>
```

### ベクトルレジスタ

次に、ベクトルを次のように宣言します：

```cpp
aie::vector<T, vec_factor> my_vector
```

* T - `int16_t`などのデータ型
* vec_factor - ベクトル化ファクタまたはベクトルサイズ（例：32）

ベクトルのサイズは型によって異なります。たとえば、AIE2の標準ベクトルレジスタは**512ビット**です。`int16_t`の場合、1つの512bベクトルレジスタに32個格納できます。これを他のサポートされているデータ型に拡張すると、次の簡略表があります：

| データ型 | ベクトルサイズ |
|-----------|-------------|
| int32_t   | 16 |
| int16_t   | 32 |
| int8_t   | 64 |
| int4_t   | 128 |

サポートされているベクトルのより完全な表は、AIE APIユーザーガイドの[こちら](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/group__group__basic__types.html)にあります。リストされているデータ型×ベクトルサイズが512ビットより大きい場合は、1つではなく2つ以上のベクトルレジスタに格納されることに注意してください。

### ベクトルロード

ローカルL1メモリから`aie::load_v`関数を使用してベクトルレジスタをロードできます。次のように定義されます：

```cpp
      T *__restrict pA1 = a;

      aie::vector<T, vec_factor> A0 = aie::load_v<vec_factor>(pA1);
```

ここでは、`__restrict`を使用してポインタを修飾し、そのポインタが基礎となるオブジェクトにアクセスする唯一のものであることを示します。これにより、ポインタエイリアシングの可能性が排除され、コンパイラによるより優れた最適化が可能になります。

ベクトルロードには、`aie::vector`宣言で使用されるものと一致するテンプレート引数`vec_factor`があります。

### ベクトル乗算とストア

最後に、ベクトルとスカラーを引数として受け取る`aie::mul`呼び出しに到達し、結果を次のように指定されたアキュムレータレジスタに格納します：

```cpp
      aie::accum<acc32, vec_factor> cout
```

この場合のアキュムレータデータ型は、32個の32ビットアキュムレータです。ベクトルストア関数`aie::store_v`を使用して、アキュムレータから計算された結果をローカルメモリに格納します。`int32_t`データ型の入力引数を使用した乗算または加算には、16個の出力レーンしかない、より大きな64ビットアキュムレータ（`acc64`）が必要になることに注意してください。

```cpp
      T *__restrict pC1 = c;

      aie::store_v(pC1, cout.template to_vector<T>(0));
```

ここで、アキュムレータ型は`.template to_vector<T>(0)`呼び出しを使用してベクトルレジスタにシフト-ラウンド-飽和できます。ここで`T`はベクトルレジスタ型で、単一の整数引数`(0)`はシフト量です。

ベクトルブロック全体は次のようになります：

```cpp
template <typename T>
void scale_vectorized(T *a, T *c, int32_t factor, const int32_t N) {
  event0();
  constexpr int vec_factor = 32;
  T *__restrict pA1 = a;
  T *__restrict pC1 = c;
  const int F = N / vec_factor;
  T fac = factor;

  AIE_PREPARE_FOR_PIPELINING
  AIE_LOOP_MIN_ITERATION_COUNT(16)
  for (int i = 0; i < F; i++)
  {
      aie::vector<T, vec_factor> A0 = aie::load_v<vec_factor>(pA1);
      pA1 += vec_factor;
      aie::accum<acc32, vec_factor> cout = aie::mul(A0, fac);
      aie::store_v(pC1, cout.template to_vector<T>(0));
      pC1 += vec_factor;
  }
  event1();
}
```

この例では、ベクトル化戦略は比較的単純でした。値のベクトルを反復処理して単一のスカラー乗算を行う代わりに、入力値のベクトルをロードし、より小さいループ（ベクトル化ファクタで除算）を反復処理してAIE API関数を使用してベクトル×スカラー演算を実行し、結果のベクトルをローカルメモリに戻します。

> **注意** - AIE APIは、世代固有の効率的な低レベルイントリンシクスに変換される型と演算を提供する、C++ヘッダーオンリーライブラリとして実装されたポータブルプログラミングインターフェースです。AIEカーネルは、これらの低レベルC++イントリンシクスで直接プログラムすることもできます：[AIE1 Intrinsics User Guide - v2023.2](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_intrinsics/intrinsics/index.html)および[AIE2 Intrinsics User Guide - v2023.2](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_ml_intrinsics/intrinsics/index.html)

## ベクトル化演習

1. ベクトルスカラーデザインのトレースを見てみましょう。まず、[vector_scalar_mul design](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)を編集して、[vector_scalar_mul.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/vector_scalar_mul.py)ソースファイルで`vectorized=False`に設定します。[vector_scalar_mul.py](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/vector_scalar_mul.py)ソースコードで、カーネル関数のスカラーバージョンを選択しました。また、環境変数`int_bit_width=32`を`makefile`コマンドに渡すことで、デザインの32ビット整数バージョンをビルドします（`make int_bit_width=32 trace`を実行）。このmakefile引数は、デザインコード（`vector_scalar_mul.py`）とホストコード（`test.cpp`）のデータ型とバッファサイズをカスタマイズするためにmakefileで定義されています。トレースコンパイルが完了したら、https://ui.perfetto.dev で`trace_vector_scalar_mul.json`を開き、`event 0`と`event 1`の間のデルタを測定します。Perfetto波形では、1 usが1クロックサイクルに等しいことに注意してください。何サイクル測定しましたか？
   <details>
   <summary>答えを見る</summary>
   約12,297サイクル
   </details>

    `vector_scalar_mul`の例では、`python/utils/get_trace_summary.py`を呼び出して、生成されたjsonファイルを自動的に分析し、`event 0`と`event 1`の間のデルタを測定し、カーネル呼び出しの数、および最初/最小/平均/最大サイクル数を提供していることに気付くかもしれません。これは、シングルコア設計のカーネル性能を要約するための便利なユーティリティです。

2. 次に、`vectorized=True`に変更してベクトル化を有効にします。ただし、その効果を確認するために、最初にプラグマガイド付き最適化を無効にします。[scale.cc](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/aie_kernels/aie2/scale.cc)で、`for loop`の前にある`AIE_PREPARE_FOR_PIPELINING AIE_LOOP_MIN_ITERATION_COUNT(16)`という行をコメントアウトします。**注意** 次にそのケースをテストするため、一般的なテンプレートと`int32_t`テンプレートの特殊化の両方を編集してください。次に、コンパイルを再実行します（`make clean; make int_bit_width=32 trace`）。再び`event 0`と`event 1`の間のデルタを測定します。今度は何が表示されますか？
   <details>
   <summary>答えを見る</summary>
   約1490サイクル
   </details>

    かなりの改善です。計算レイテンシが約8倍削減されました。ただし、ベクトルコードでさらに最適化できることがあり、それには最適化プラグマが含まれます。

3. [scale.cc](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/aie_kernels/aie2/scale.cc)に戻り、`AIE_PREPARE_FOR_PIPELINING AIE_LOOP_MIN_ITERATION_COUNT(16)`の行のコメントを外してこれらのプラグマを有効にします。次に、コンパイルを再実行します（`make clean; make int_bit_width=32 trace`）。再び`event 0`と`event 1`の間のデルタを測定します。今度は何が表示されますか？
   <details>
   <summary>答えを見る</summary>
   339サイクル
   </details>

    今、本当に大きな節約が見られます（さらに約4倍の節約、またはスカラーバージョンと比較して約36倍）。追加した行は、コンパイラが最適なスケジュールを見つけるのに役立ちます。カーネルループの場合、`AIE_PREPARE_FOR_PIPELINING`と`AIE_LOOP_MIN_ITERATION_COUNT(16)`が特に役立ちます：
    * `AIE_PREPARE_FOR_PIPELINING` - 最内ループで使用して、ソフトウェアパイプライニングを有効にするようコンパイラに指示します。これは、後続のループ最適化プラグマを有効にするために必要です。
    * `AIE_LOOP_MIN_ITERATION_COUNT(MIN)` - 非常に役立つプラグマです。これは、このループが持つと予想される最小反復回数をコンパイラに伝えます。最小反復回数と最大反復回数の両方を指定したい場合は、`, AIE_LOOP_RANGE(MIN,MAX)`を使用できます。多くの場合、サイズに基づいてループ境界をパラメータ化し、ループサイズがconstとして宣言されていても、それは依然としてランタイムで計算される値です。このプラグマでMIN値を指定すると、反復回数がわかるため、スケジューラがそのガイドとなり、最悪の場合の1ではなく、その数に対してループ命令を適切にスケジュールできるため、特に役立ちます。

4. 最後に、ベクトル化を有効にし、カーネルコードの最適化プラグマを有効にした状態で、デザインの16ビット整数バージョンでコンパイルを再実行します。これは`vector_scalar_mul`デザインのデフォルト設定です（`make clean; make trace`）。再び`event 0`と`event 1`の間のデルタを測定します。今度は何が表示されますか？
   <details>
   <summary>答えを見る</summary>
   78サイクル
   </details>

    反復ごとに2倍のデータを処理でき、反復ごとに必要なベクトル乗算が少なくて済むため、さらに4倍の改善が見られ、スカラーバージョンから合計約157倍の改善が得られました。

## 最適化 - アーキテクチャを考慮したコーディング

この時点で、AIEハードウェアをより有効に活用するためにコードをベクトル化し、大幅な性能向上を確認しましたが、デザインは完全に最適化されていますか？強力なAIEハードウェアをそのフルポテンシャルまで使用したかどうかをどのように知るのでしょうか？これには、基礎となるAIEアーキテクチャのより深い理解と、ハードウェアを念頭に置いた性能のためのコーディングが必要です。この次のセクションでは、Ryzen™ AI NPUの中核にある**AIE2**（別名AIE-ML）に焦点を当てます。AIE2はMLワークロードに最適化されています。つまり、行列乗算スタイルの計算のような乗算累算演算がハードウェアを最もよく活用します。また、ベクトルスカラー乗算の例を続けて探索を開始します。すべての最適化を活用するのに十分な計算量を公開していませんが、最適なデザインをコーディングするために必要な設計上の考慮事項を理解するための良い出発点を提供します。

### ベクトルユニット - ロード

コードをさらに最適化する最初のステップは、AIEベクトルユニットの全体像を持つことです。これは[AIE-MLアーキテクチャマニュアル（am020）](https://docs.amd.com/r/en-US/am020-versal-aie-ml/Fixed-Point-Vector-Unit)にあります。以下は、マニュアルからのベクトルユニットの図です。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/aie-ml_vector_unit.png" title="AIE-ML Vector Unit." height=450>

ご覧のとおり、ベクトルレジスタは2つの並列ロードユニットからロードされ、それぞれがローカルL1メモリからクロックサイクルあたり256ビットをロードできます。12個の512ビットベクトルレジスタがあり、各Permuteブロックに供給され、最終的にMultiplierブロックに供給されます。したがって、クロックサイクルあたり2×256ビットの並列ロードの観点から常に考えることが重要です。たとえば、計算を行うためにクロックあたり2048ビットのデータをロードしようとすると、複数のサイクルが必要になるため、効率が低下します。もう1つの重要な注意点は、ロードは異なるL1メモリバンクから行う必要があることです。そうしないと、バンク競合が発生します。バンク競合のペナルティは1サイクルですが、最適な性能が低下します。

### ベクトルユニット - 乗算と加算（MAC）

データがロードされて並べ替えられると、幅広いAIEデータ型をサポートするMultiplierブロックに渡されます。乗算結果は、オプションのポスト加算ステップ（行列乗算で非常に一般的）を通過してから、最終的にアキュムレータレジスタに格納されます。9個の512ビットアキュムレータレジスタがあります。アキュムレータレジスタはより大きいため、データ精度を維持できます。適切に最適化されたコードは、サイクルごとに1つのベクトルMAC（VMAC）をスケジュールするよう努力します。

### ベクトルユニット - SRSとストア

データが計算されると（1サイクルで、または複数のサイクルにわたって累算されて）、結果はStore Unitを介してローカルL1メモリに書き戻すことができます。これは2つのロードユニットを反映していますが、ストアユニットは1つだけです。アキュムレータレジスタとベクトルレジスタまたはローカルL1メモリの間のブリッジングには、SRSユニット（shift-round-saturate）を利用します。これは、多数の設定可能な丸めおよび飽和モードを使用してシフト、丸め、飽和を行います。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/aie-ml_srs_ups.png" title="AIE-ML SRS UPS Unit." height=230>

SRSパスは上図の右側にあり、相関するパスであるUpshift（UPS）パスは左側にあります。Upshiftはベクトルレジスタからアキュムレータレジスタにデータを移動します。

### ベクトルユニット - Shift/Shuffle/Adderパス

最後に、シフト、シャッフル、単純な加算、比較、およびその他の多数のベクトル関数を実行する追加の並列処理パスがあります。このパスは、メイン整数ベクトルデータパスと並行して実行され、VMACデータパスを必要とせずに前述の関数を実行するようにタスク化される場合があります。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/aie-ml_shift_adder_path.png" title="AIE-ML Shift Adder Unit." height=200>

この処理データパスと、データがローカルメモリとの間でロードおよび格納される方法を念頭に置いておくことは非常に役立ちます。次のステップは、アプリケーションで理想的な性能にどれだけ近いかを確認し、結果をより詳細に調べて、改善できる場所をよりよく理解することです。

### 乗算器の利用効率

アーキテクチャをよりよく理解したので、ハードウェア効率をより詳しく見てみましょう。次の図は、説明したさまざまなAIEアーキテクチャブロックと、一般化された計算の表を示しています。

<img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/aie_compute_details1.png" title="AIE Compute Details." height=450>

> **注意** - 行列乗算モードの表は、AIE APIユーザーガイドの[こちら](https://www.xilinx.com/htmldocs/xilinx2023_2/aiengine_api/aie_api/doc/group__group__mmul.html)にあります。さまざまなビット精度の合計MAC数を確認する別の方法は、[AM020仕様](https://docs.amd.com/r/en-US/am020-versal-aie-ml/Functional-Overview)の`Table: Supported Precision Width of the Vector Data Path`です。

この表は、16ビット×16ビット計算の場合、サイクルあたり64個のMACが利用可能であることを示しています。ただし、これらのMACは行列乗算（付随するポスト加算ステップを含む）を対象としています。実際には、32個のアキュムレータレーンが利用可能です。つまり、要素ごとの演算の場合、サイクルあたり32個のMACしか使用できません。

#### MAC効率

この情報とベクトルスカラー乗算の例を使用すると、カーネルへの各呼び出しが1024個の16ビットデータの配列を渡すことがわかります。32個のMACが利用可能で、`vector_factor`は32であるため、32 MACsパークロック要素ごとのベクトルMAC構成を考えると、このデータ量を処理するために理想的には1024 / 32 = 32サイクルが必要です。カーネルの最終的な最適化されたサイクルカウントは72サイクルで、理想的なサイクル数の約2倍です。

合計MAC効率は、（MACスケジュール効率）×（クロックあたりのMAC利用効率）の積です。
* （MACスケジュール効率または理想的なMACサイクル）/（実際のMACサイクル）、例：32/ 72 = 44%
* （クロックあたりのMAC利用効率または使用されたMACの数）/（利用可能なMACの総数）、例：32/ 64 = 50%
したがって、合計MAC効率は44% × 50% = 22%です。

この結果をファイルしておきますが、ロード/ストア帯域幅の観点からアルゴリズムを見てみましょう。

#### ロード/ストア帯域幅効率

32個のint16値のベクトルにスカラーを乗算して処理するには、スカラーロードを無視して、ベクトルのみに焦点を当てましょう。32 int16 = 512ビットで、2×256ビットロードまたは2サイクルかかります。データがバンク間でインターリーブされている場合、1サイクルで実行できる可能性があります。また、2×256ビットを格納する必要があり、ストアユニットが1つしかないため、2サイクルかかります。つまり、サイクルごとにVMACを実行できたとしても、入力をロードするために2サイクル、出力を格納するために2サイクルが必要です。この2サイクル要件に基づいて、データサイズの最小サイクルが64サイクルであるため、最適化されたベクトル結果が72である理由がわかります。残りの6サイクルは、ループプリアンブル、ループポストアンブル、および関数の初期化とクリーンアップのオーバーヘッドです。

#### データルーティング効率

ロード/ストア帯域幅は、16ビットベクトルスカラー乗算の例では、計算にとって既にボトルネックです。しかし、ストリームとDMAを介したデータ移動についてはどうでしょうか。1024チャンクの16ビットデータ、または512個の32ビット量を処理する必要があります。ストリームスイッチが32ビット粒度でデータを移動するため、データをL1にロードし、L1からL2/L3にデータを移動するには512サイクルが必要です。

#### ハードウェア効率の要約

| コンポーネント | サイクル数 | 効率 |
|-----------|-------------|------------|
| MAC       | 72          | 22%        |
| ロード/ストア| 64          | 50% / 100% |
| DMA       | 512         | 100%       |

この表を見ると、データ移動が全体的なカーネルのボトルネックであることがすぐにわかります。

## 最適化演習 - パート1

1. 最終的な最適化されたコードを再実行して、結果の波形を見てください。

    <img src="https://raw.githubusercontent.com/Xilinx/mlir-aie/v1.1.1/programming_guide/assets/aie_vector_scalar_ml_opt1.png" title="AIE Compute Details." height=250>

    PortRunning0とPortRunning1のブロックにマウスを合わせると、チャンクあたりの測定サイクル数はどれくらいですか？
   <details>
   <summary>答えを見る</summary>
   512サイクル
   </details>

   これは期待どおりです。ただし、計算と比較してデータ移動がどれほど支配的であるかが波形から明らかであることに注意してください。

2. デザインがデータ移動と計算の間で不均衡であることは既にわかっています。計算に72サイクル、データ移動に512サイクルがあります。[行列乗算の例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/single_core)を見て、改善できるかどうかを確認しましょう。説明では、カーネルの各反復がデフォルトで64x64x64のMxKxN値に対して構成されており、262,144個のMACが得られることが説明されています。`int16_t`データ型を使用しており、クロックあたり64個のMACがある場合、理想的なケースは何サイクルかかりますか？
   <details>
   <summary>答えを見る</summary>
   2048サイクル = 262,144 / 64
   </details>

   AとB行列はそれぞれ64x64 × `int16_t`で、ストリームスイッチチャネルは32ビット幅です。計算タイルにデータを移動するのに何サイクルかかりますか（AとBは別々のチャネルを介して並行して移動できることに注意してください）。
   <details>
   <summary>答えを見る</summary>
   2048サイクル = 64x64 / 2
   </details>

3. この例は、計算とデータ移動の間で完全にバランスが取れているはずです！[行列乗算の例](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/single_core)に移動して、トレースビルドを実行します（`make clean; make -f Makefile.chess use_placed=1 trace`）。次に、生成された波形json（`trace_mm.json`）を開き、最初の実行で`event 0`と`event 1`の間のデルタを測定します。どんな値が得られましたか、そしてそれは理想にどれほど近いですか？
   <details>
   <summary>答えを見る</summary>
   約2535サイクル、これは2048の80%です
   </details>

   計算サイクルとデータ移動サイクルがはるかに近く一致していることがわかるはずです！

## 深く掘り下げる - マイクロコードの検査

[vector_scalar_mul design](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/vector_scalar_mul/)の結果をもう一度見てみましょう。また、1ステップ戻って`AIE_PREPARE_FOR_PIPELINING AIE_LOOP_MIN_ITERATION_COUNT(16)`をコメントアウトし、コンパイルを再実行します（`make clean; make trace`）。

この時点で、実際に逆アセンブリコードを見ることができます。逆アセンブリは、カーネルプログラムを実行するためにAIEが実行する正確な命令のスケジュールです。逆アセンブリを取得するには、生成されたオブジェクトまたはelfファイルで`llvm-objdump`を実行できます。

```bash
<mlir-aie>/ironenv/lib/python<ver>/site-packages/llvm-aie/bin/llvm-objdump -dr build/core_0_2.elf > disassembly_0_2.txt
```

elfファイルの場合、命名には`core`名が含まれ、コアの2つの数字はそれぞれ列と行の位置を示します。したがって、デザインに複数のコアがある場合、各コアには逆アセンブルできる独自の`.elf`ファイルがあります。逆アセンブリファイルを生成して開くと、多くの情報が表示されます。コメント行の前には.があります。他の行は命令であり、次のように構成されています：

```
命令行番号 ---- エンコードされた命令 ---- 1つ以上のスロットのISAコマンド
```

| ISAコマンドの例 | 説明 |
|----------------------|-------------|
| NOP .. | No op |
| JL #XXX | 命令行番号#にジャンプしてリンク |
| MOV r1, r2 | レジスタ値をr2からr1に移動 |
| LD .. | スカラーロード |
| ST ..  | スカラーストア |
| VLDA | ベクトルロードユニットA |
| VLDB | ベクトルロードユニットB |
| VMUL ..  | ベクトル乗算 |
| VMAC ..  | ベクトル乗算累算 |
| VST .. | ベクトルストア |
| VSRS .. | ベクトルSRS |
| VSHUFFLE .. | ベクトルシャッフル |

このマイクロコードを完全に分析して理解することは、このプログラミングガイドの範囲を超えていますが、特に3種類のコメントでラベル付けされたこのマイクロコードの主要部分に焦点を当てます：

`<8桁の数字> <label>` ここで`<label>`は`<main>`または`<vector_scalar_mul_vector>`のような関数名です - 関心のある関数の開始。

`<8桁の数字> <.LBB?_?>:` - ゼロオーバーヘッドループの開始。

`<8桁の数字> <.L_LEnd?>` - ゼロオーバーヘッドループの終了。
> **注意** このラベルの後の行がループ内の最後の行であり、`<.LBB?_?>`と`.L_LEnd?`の間の行だけではありません。一般に、ラベルはラベルの後の行に対するものです。

例でこれをより詳しく調べてみましょう。

## 最適化演習 - パート2

1. 戻ってプラグマ行（`AIE_PREPARE_FOR_PIPELINING AIE_LOOP_MIN_ITERATION_COUNT(16)`）を再度コメントアウトし、ビルドを再実行します（`make clean; make trace`）。逆アセンブラを実行して`disassembly_0_2.txt`を開き、ファイルを見てください。

    `vector_scalar_mul_vector`を検索します。次に、その後の最初の`LBB`行が表示されるまでスクロールダウンします。次の`LBB`または`L_LEnd`行に到達するまでの行数をカウントします。次の行が`L_LEnd`の場合は、合計カウントに1を追加してください。この内部ループには何行ありますか？
   <details>
   <summary>答えを見る</summary>
   9サイクル
   </details>

2. 次に、各行を見て、`VMUL`または`VMAC`を含む行がいくつあるかをカウントします。どんな数字が得られますか？
   <details>
   <summary>答えを見る</summary>
   VMULは1つだけ
   </details>

3. 得られた数値は、アルゴリズムの最内ループがどれほど最適化されているかの大まかなアイデアを提供します。この場合、9サイクル中1つのVMACがあり、MAC利用率は約11%です。内部ループが11サイクルかかり、32回反復する場合、このバージョンは何サイクルかかり、測定されたサイクルカウントにどれだけ近いですか？
   <details>
   <summary>答えを見る</summary>
   11*32=352サイクル、測定された約309サイクルのうち。かなり近いです。オーバーヘッドは約15サイクルです
   </details>

4. 次に、戻ってプラグマ行を再度コメント解除し、ビルドとクリーンアップスクリプトを再実行します（`make clean; make trace; <mlir-aie>/ironenv/lib/python<ver>/site-packages/llvm-aie/bin/llvm-objdump -dr build/core_0_2.elf > disassembly_0_2.txt`）。再び`vector_scalar_mul_vector`を検索し、内部ループの行数と`VMUL/VMAC`行を再度カウントします。何が見えますか？
   <details>
   <summary>答えを見る</summary>
   内部ループ行は2行。VMULは1つ。
   </details>

   これは、ベクトルストアのために内部ループが2に制限されているという手計算と一致します。

---

**注意**: より詳細な情報、完全なコード例、および高度な最適化技術については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4c)を参照してください。
