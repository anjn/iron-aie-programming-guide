# クイックリファレンス

このページでは、IRON AI Engineプログラミングで頻繁に使用される重要なAPI要素とコンセプトをまとめています。

## 主要なPython API

### Device クラス

```python
from aie.dialects.aie import *
```

### ObjectFIFO

データ移動の基本構造

```python
# 基本的な使い方
objectfifo(name, producer_tile, consumer_tiles, depth, dtype)
```

### Tile設定

```python
# コンピュートタイル
tile(col, row)

# メモリタイル
mem_tile(col)
```

## データ型

- `i8`, `i16`, `i32`: 整数型
- `f32`, `bf16`: 浮動小数点型
- `vector<N x type>`: ベクトル型

## よく使うパターン

### 基本的なデータフロー

1. ObjectFIFOの定義
2. プロデューサとコンシューマの設定
3. カーネル関数の実装
4. データ転送の設定

---

**翻訳作業中**: このセクションの詳細な翻訳は進行中です。最新の情報は[公式ドキュメント](https://github.com/Xilinx/mlir-aie/blob/main/programming_guide/quick_reference.md)を参照してください。
