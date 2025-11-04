# Section 4a - タイマー

AMD AI Engineアクセラレータでのアプリケーション性能測定にタイマーを使用できます。このタイマーは、OS、カーネルドライバ、AIE配列への作業のディスパッチ、AIEコアでの計算など、ソフトウェアスタック全体のオーバーヘッドを含む「ウォールクロック」時間を測定します。このオーバーヘッドは含まれますが、複数回の反復を実行する場合や計算集約的なワークロードを実行する場合に、意味のある実用的な性能上限データを提供します。

## アプリケーションタイマー - test.cppの修正

アプリケーションタイマーを追加するには、[test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4a/test.cpp)を修正します。chronoライブラリを使用して、カーネル関数の実行前後で時刻をキャプチャします。単一のカーネル実行の場合、次のようになります：

```cpp
    auto start = std::chrono::high_resolution_clock::now();
    unsigned int opcode = 3;
    auto run = kernel(opcode, bo_instr, instr_v.size(), bo_inA, bo_inFactor, bo_outC);
    run.wait();
    auto stop = std::chrono::high_resolution_clock::now();

    float npu_time = std::chrono::duration_cast<std::chrono::microseconds>(stop - start).count();
    std::cout << "NPU time: " << npu_time << "us." << std::endl;
```

## 複数回の反復

複数回の反復を使用すると、定常状態のカーネル実行時間をよりよくキャプチャできます。基本的なforループは次のようになります：

```cpp
  unsigned num_iter = n_iterations + n_warmup_iterations;
  for (unsigned iter = 0; iter < num_iter; iter++) {
    <... kernel run code ...>
  }
```

### ウォームアップ反復

初期のカーネル実行はしばしば起動オーバーヘッドを示すため、いくつかのウォームアップ反復を実行することをお勧めします。これにより、代表的な定常状態のタイミングを達成し、外れ値を減らすことができます。ウォームアップ反復は平均実行時間にカウントしないでください。

```cpp
  for (unsigned iter = 0; iter < num_iter; iter++) {
    <... kernel run code ...>
    if (iter < n_warmup_iterations) {
      /* Warmup iterations do not count towards average runtime. */
      continue;
    }
    <... verify and measure timers ...>
  }
```

### タイマーデータの蓄積

各反復でタイマーデータを蓄積して、平均、最小、最大の実行時間を計算できます：

```cpp
  for (unsigned iter = 0; iter < num_iter; iter++) {
    <... kernel run code, warmup conditional, verify, and measure timers ...>
    npu_time_total += npu_time;
    npu_time_min = (npu_time < npu_time_min) ? npu_time : npu_time_min;
    npu_time_max = (npu_time > npu_time_max) ? npu_time : npu_time_max;
  }
```

### 結果の計算と出力

すべての反復を実行した後、結果を計算して出力します：

```cpp
  std::cout << "Avg NPU time: " << npu_time_total / n_iterations << "us." << std::endl;
  std::cout << "Min NPU time: " << npu_time_min << "us." << std::endl;
  std::cout << "Max NPU time: " << npu_time_max << "us." << std::endl;
```

カーネルのMAC（乗算累算）数がわかっている場合は、GFLOPSなどの追加の性能メトリクスを計算できます。詳細については、[matrix_multiplication/test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_examples/basic/matrix_multiplication/test.cpp#L170)を参照してください。

## 演習

1. [test.cpp](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4a/test.cpp)をビルドして実行します（`make`と`make run`）。`Avg NPU time`の値はどの範囲ですか？
   <details>
   <summary>答えを見る</summary>
   答えは300-600usのどこかです
   </details>

2. `n_iterations`を1から10に変更して再度実行します。`Avg NPU time`の値はどうなりましたか？
   <details>
   <summary>答えを見る</summary>
   答えは依然として300-600usのどこかですが、以前とは異なる可能性が高いです
   </details>

3. `n_warmup_iterations`を0から4に変更します。`Avg NPU time`の値はどうなりましたか？
   <details>
   <summary>答えを見る</summary>
   今回は、300-400usの狭い範囲が表示されます
   </details>

4. `n_iterations`を10から100に変更します。`Avg NPU time`の値はどうなりましたか？
   <details>
   <summary>答えを見る</summary>
   今回は、200-300usのより低い平均範囲が表示されます
   </details>

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4a)を参照してください。
