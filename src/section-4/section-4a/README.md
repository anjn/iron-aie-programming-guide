# Section 4a - タイマー

## 概要

このセクションでは、AMD AI Engineアクセラレータでのアプリケーション性能測定にタイマーを使用する方法について説明します。

## 核心概念

### アプリケーションタイマーの目的

「ウォールクロック」時間を測定することで、以下を含むソフトウェアスタック全体のオーバーヘッドをキャプチャします：

- OS とのやり取り
- カーネルドライバ通信
- AIE配列への作業のディスパッチ
- AIEコアでの計算

オーバーヘッドは含まれますが、複数回の反復や計算集約的なワークロードを実行する場合に意味のある実用的な性能上限データを提供します。

### 実装アプローチ

開発者は、C++のchronoライブラリを使用してタイミングを実装します：

```cpp
#include <chrono>

// カーネル関数実行前
auto start = std::chrono::high_resolution_clock::now();

// カーネル実行
kernel_function();

// カーネル関数実行後
auto stop = std::chrono::high_resolution_clock::now();

// マイクロ秒単位で期間を計算
auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
    stop - start
);

std::cout << "実行時間: " << duration.count() << " μs" << std::endl;
```

## 測定戦略

### 1. 単一実行ベースライン

**目的:** 初期の性能推定を取得

**実装:**

```cpp
auto start = std::chrono::high_resolution_clock::now();
kernel.run();
auto stop = std::chrono::high_resolution_clock::now();

auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
    stop - start
);
std::cout << "単一実行時間: " << duration.count() << " μs" << std::endl;
```

### 2. 複数回反復ベンチマーク

**目的:** 変動を平滑化し、定常状態の性能をキャプチャ

**実装:**

```cpp
const int num_iterations = 100;
std::vector<long> timings;

for (int i = 0; i < num_iterations; i++) {
    auto start = std::chrono::high_resolution_clock::now();
    kernel.run();
    auto stop = std::chrono::high_resolution_clock::now();

    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
        stop - start
    );
    timings.push_back(duration.count());
}

// 統計の計算
long sum = std::accumulate(timings.begin(), timings.end(), 0L);
long avg = sum / num_iterations;
long min = *std::min_element(timings.begin(), timings.end());
long max = *std::max_element(timings.begin(), timings.end());

std::cout << "平均: " << avg << " μs" << std::endl;
std::cout << "最小: " << min << " μs" << std::endl;
std::cout << "最大: " << max << " μs" << std::endl;
```

**利点:**
- 統計的に意味のある結果
- 外れ値の識別
- より信頼性の高い性能データ

### 3. ウォームアップ反復

**目的:** 起動オーバーヘッドを除外

初期のカーネル実行は、しばしば起動オーバーヘッドを示します。方法論では、代表的な定常状態のタイミングを達成し、外れ値を減らすために、予備反復（測定にカウントされない）を実行することを推奨します。

**実装:**

```cpp
const int warmup_iterations = 4;
const int num_iterations = 10;

// ウォームアップ（測定しない）
for (int i = 0; i < warmup_iterations; i++) {
    kernel.run();
}

// 実際の測定
std::vector<long> timings;
for (int i = 0; i < num_iterations; i++) {
    auto start = std::chrono::high_resolution_clock::now();
    kernel.run();
    auto stop = std::chrono::high_resolution_clock::now();

    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
        stop - start
    );
    timings.push_back(duration.count());
}
```

**理由:**
- 最初の実行はキャッシュが冷たい
- ドライバの初期化オーバーヘッド
- その他の起動コスト

### 4. 拡張メトリクス

実行時間統計に加えて、開発者はカーネルのMAC（乗算累算）数がわかっている場合、追加の性能指標を計算できます。

**GFLOPS計算例:**

```cpp
// カーネル情報
const long ops_per_iteration = 2048;  // 例：1024回の乗算累算 = 2048 ops
const double avg_time_sec = avg_time_us / 1e6;

// GFLOPS = (Operations) / (Time in seconds) / 1e9
double gflops = (ops_per_iteration * num_iterations) / avg_time_sec / 1e9;

std::cout << "性能: " << gflops << " GFLOPS" << std::endl;
```

**その他の有用なメトリクス:**
- スループット（要素/秒）
- 帯域幅使用率（GB/s）
- 効率（理論的ピークに対する%）

## 実践演習

ドキュメントでは、テストプログラムの構築、実行、反復を通じて、4つの段階的な演習をユーザーに案内します：

### 演習1: 基本的な単一実行

```bash
make
make run
```

**学習内容:** 基本的なタイミング測定

### 演習2: 複数回実行

コードを修正して10回反復を実行：

```cpp
const int num_iterations = 10;
for (int i = 0; i < num_iterations; i++) {
    // タイミング測定
}
```

**学習内容:** 統計的な信頼性の向上

### 演習3: ウォームアップ追加

4回のウォームアップサイクルを追加：

```cpp
const int warmup_iterations = 4;
// ウォームアップループ
for (int i = 0; i < warmup_iterations; i++) {
    kernel.run();
}
```

**学習内容:** 起動オーバーヘッドの除外

### 演習4: 詳細な統計

平均、最小、最大を計算：

```cpp
// 統計計算コードの追加
long avg = sum / num_iterations;
long min = *std::min_element(timings.begin(), timings.end());
long max = *std::max_element(timings.begin(), timings.end());
```

**学習内容:** より詳細な性能分析

## ベストプラクティス

### 測定の精度向上

1. **十分な反復回数**: 最低10回、理想的には100回以上
2. **ウォームアップ**: 常にウォームアップ反復を含める
3. **環境の安定性**: 他のプロセスの影響を最小化
4. **一貫性のある条件**: 同じ入力サイズとパターンを使用

### 結果の解釈

- **平均**: 典型的な性能を表す
- **最小**: 理想的な条件下の性能
- **最大**: 最悪ケースのシナリオ
- **標準偏差**: 変動の尺度

### よくある落とし穴

- ウォームアップなしでの測定
- 単一実行のみに依存
- システム負荷を考慮しない
- デバッグビルドでの測定

---

**注意**: より詳細な情報と完全なコード例については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4a)を参照してください。
