# Section 4c - カーネルのベクトル化と最適化

## 概要

このドキュメントは、AIE（AI Engine）プロセッサのカーネルベクトル化技術と性能最適化戦略について説明し、特にベクトルスカラー乗算の例と行列乗算ワークロードに焦点を当てています。

## AIE APIとベクトルプログラミング

### ヘッダーのインクルード

```cpp
#include <aie_api/aie.hpp>
```

AIE APIは低レベルイントリンシクスを抽象化し、より使いやすいインターフェースを提供します。

### ベクトルレジスタの宣言

ベクトルレジスタは`aie::vector<T, vec_factor>`として宣言されます。ベクトルファクタはデータ型に依存します：

| データ型 | ベクトルファクタ | 要素数 |
|---------|-----------------|--------|
| `int32_t` | 16 | 16要素 |
| `int16_t` | 32 | 32要素 |
| `int8_t` | 64 | 64要素 |
| `int4_t` | 128 | 128要素 |

**重要:** AIE2アーキテクチャでは、すべてのベクトルが512ビットレジスタを使用します。

### ベクトル化の基本例

```cpp
// スカラーバージョン
void scalar_mult(int32_t* a, int32_t* c, int32_t factor, int32_t N) {
    for (int i = 0; i < N; i++) {
        c[i] = a[i] * factor;
    }
}

// ベクトル化バージョン
void vector_mult(int32_t* __restrict a, int32_t* __restrict c,
                 int32_t factor, int32_t N) {
    const int vec_factor = 16;  // int32_tは16要素
    aie::vector<int32_t, vec_factor> vec_a;
    aie::accum<acc32, vec_factor> acc;

    for (int i = 0; i < N; i += vec_factor) {
        vec_a = aie::load_v<vec_factor>(a + i);
        acc = aie::mul(vec_a, factor);
        aie::store_v(c + i, acc.template to_vector<int32_t>(0));
    }
}
```

## ベクトル化の構成要素

### 1. ロード（Loading）

**関数:** `aie::load_v<vec_factor>()`

**`__restrict`修飾子の使用:**

```cpp
void func(int32_t* __restrict a, int32_t* __restrict c) {
    // __restrict はコンパイラにポインタがエイリアスしないことを保証
    // これにより最適化が可能になる
}
```

**例:**

```cpp
// vec_factor要素をロード
aie::vector<int32_t, 16> vec_a = aie::load_v<16>(a_ptr);
```

### 2. 計算（Computation）

**関数:** `aie::mul()`

ベクトルスカラー演算を実行し、結果をアキュムレータに格納します。

```cpp
aie::accum<acc32, 16> acc;
acc = aie::mul(vec_a, scalar_factor);
```

**アキュムレータの利点:**
- より高い精度（32ビット整数の場合、80ビットアキュムレータ）
- オーバーフローの回避
- 複数演算の累積

### 3. ストア（Storage）

**変換と格納:**

```cpp
// アキュムレータをベクトルに変換
auto result = acc.template to_vector<int32_t>(0);

// メモリに格納
aie::store_v(c_ptr, result);
```

## コンパイラプラグマ

2つの最適化ディレクティブが性能に大きな影響を与えます：

### 1. ソフトウェアパイプライニング

```cpp
AIE_PREPARE_FOR_PIPELINING
```

**目的:** 最内ループでソフトウェアパイプライニングを有効化

**使用例:**

```cpp
for (int i = 0; i < N; i += vec_factor)
AIE_PREPARE_FOR_PIPELINING
{
    vec_a = aie::load_v<vec_factor>(a + i);
    acc = aie::mul(vec_a, factor);
    aie::store_v(c + i, acc.template to_vector<int32_t>(0));
}
```

### 2. 最小反復回数の指定

```cpp
AIE_LOOP_MIN_ITERATION_COUNT(N)
```

**目的:** コンパイラに最小ループ反復回数を通知し、命令スケジューリングを改善

**使用例:**

```cpp
AIE_LOOP_MIN_ITERATION_COUNT(64)
for (int i = 0; i < N; i += vec_factor) {
    // ループ本体
}
```

### 性能改善

演習では以下の改善を実証します：

1. **ベクトル化のみ**: 約8倍の改善
2. **プラグマ追加**: さらに4倍の改善
3. **合計**: スカラーベースラインから**36倍**の改善

## アーキテクチャの詳細

### ベクトルユニットコンポーネント

AIE2のベクトルユニットには以下が含まれます：

**ロード/ストアユニット:**
- 2つの並列256ビットロードユニット（サイクルあたり）
- 1つのストアユニット（サイクルあたり256ビット）

**レジスタ:**
- 12個の512ビットベクトルレジスタ
- 9個の512ビットアキュムレータレジスタ

**演算ユニット:**
- SRS（Shift-Round-Saturate）ユニット（精度処理用）
- Shift/Shuffle/Adder並列処理パス

**図解:**

```
メモリ
  ↓ (2×256-bit Load)
[ベクトルレジスタ × 12]
  ↓
[演算ユニット]
  ↓
[アキュムレータ × 9]
  ↓ (SRS)
[ストアユニット]
  ↓ (256-bit Store)
メモリ
```

### MAC効率分析

**16ビット要素ごとの演算:**
- サイクルあたり32 MAC（行列演算では64）

**ベクトルスカラー例（1024個のint16値を処理）:**

| メトリクス | 値 |
|----------|-----|
| 理想計算サイクル | 32 (1024/32) |
| 測定サイクル | ~72 |
| ロード/ストア要件 | 最低2サイクル |

**計算:**
- 512ビットデータ
- 2×256ビットロード + 1×256ビットストア
- 実効サイクル = 2（最小）

## 性能ボトルネックの特定

### ワークロード分析

| コンポーネント | サイクル | 効率 |
|--------------|---------|------|
| MAC演算 | 72 | 22% |
| ロード/ストア | 64 | 50-100% |
| DMA | 512 | 100% |

**結論:** データ移動が主要なボトルネック

### 行列乗算での改善

行列乗算の例では、計算とデータ移動のバランスが向上：

- **理想計算サイクル:** 2048
- **A/B行列の並列ロード:** 2048サイクル
- **バランスの取れた設計:** 計算とデータ移動が等しい

## マイクロコード分析

### 逆アセンブリの検査

`llvm-objdump`を使用してマイクロコードを調査：

```bash
llvm-objdump -dr build/core_0_2.elf > disassembly.txt
```

### 主要命令タイプ

**ベクトルロード:**
- `VLDA`: ベクトルロードA
- `VLDB`: ベクトルロードB

**ベクトル演算:**
- `VMUL`: ベクトル乗算
- `VMAC`: ベクトル乗算累算

**ベクトルストア:**
- `VST`: ベクトルストア

**精度調整:**
- `VSRS`: ベクトルシフト-ラウンド-飽和

### ループ構造分析

**ラベル:**
- `<.LBB?_?>`: ゼロオーバーヘッドループの開始
- `<.L_LEnd?>`: ループの終了

**プラグマなしの内部ループ:**
- 約9命令
- 1つのVMUL（~11%利用率）

**プラグマ最適化後:**
- 2命令に削減
- ベクトルストア帯域幅制約により制限

### 分析例

```asm
# プラグマなし（非効率）
.LBB0_1:
    VLDA    r0, [r1]       ; ロードA
    VLDB    r2, [r3]       ; ロードB
    VMUL    r4, r0, r2     ; 乗算
    VSRS    r5, r4         ; シフト-ラウンド-飽和
    VST     [r6], r5       ; ストア
    ADD     r1, r1, #64    ; ポインタ更新
    ADD     r3, r3, #64
    ADD     r6, r6, #64
    CMP     r7, r1         ; ループ条件
    BNE     .LBB0_1

# プラグマ最適化後（効率的）
.LBB0_1:
    VMAC    r4, r0, r2     ; 乗算累算（パイプライン化）
    VST     [r6], r5       ; ストア
    ; その他の操作はパイプライン内で並列実行
```

## 性能サマリー

### 改善の内訳

**ベースライン（スカラー）:** 1.0x

**ステップ1 - ベクトル化:** 8.0x
- int16で32要素を並列処理
- ベクトルレジスタの活用

**ステップ2 - プラグマ最適化:** 4.0x（前ステップから）
- ソフトウェアパイプライニング
- 命令レベル並列性

**最終結果:** 36x（スカラーから）

**16ビット要素ごとの演算での最終性能:**
- **157倍の改善**（アーキテクチャ認識と組み合わせ）
- 最終的に**データ移動帯域幅**によって制約

### 最適化の教訓

1. **ベクトル化は不可欠**: 8倍の即座の改善
2. **コンパイラヒント重要**: プラグマでさらに4倍
3. **データ移動がボトルネック**: 計算能力よりも制約
4. **バランスの取れた設計**: 行列演算で計算とデータ移動を一致

## ベストプラクティス

### コーディング

```cpp
// 良い例
void optimized_kernel(
    int16_t* __restrict a,
    int16_t* __restrict b,
    int16_t* __restrict c,
    int32_t N
) {
    const int vec_factor = 32;
    aie::vector<int16_t, vec_factor> va, vb;
    aie::accum<acc32, vec_factor> acc;

    AIE_LOOP_MIN_ITERATION_COUNT(64)
    for (int i = 0; i < N; i += vec_factor)
    AIE_PREPARE_FOR_PIPELINING
    {
        va = aie::load_v<vec_factor>(a + i);
        vb = aie::load_v<vec_factor>(b + i);
        acc = aie::mul(va, vb);
        aie::store_v(c + i, acc.template to_vector<int16_t>(0));
    }
}
```

### データ型の選択

- **int8_t**: 最高のベクトルファクタ（64）、低精度
- **int16_t**: 良好なバランス（32）
- **int32_t**: 低いベクトルファクタ（16）、高精度
- **float**: 特殊用途、ベクトルファクタ16

### デバッグと検証

1. スカラーバージョンから開始
2. ベクトル化を追加
3. 結果を検証
4. プラグマを追加
5. 性能を測定
6. 逆アセンブリを分析

---

**注意**: より詳細な情報、完全なコード例、および高度な最適化技術については、[公式ドキュメント](https://github.com/Xilinx/mlir-aie/tree/v1.1.1/programming_guide/section-4/section-4c)を参照してください。
