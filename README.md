# IRON AIE Programming Guide（非公式日本語訳）

[![Deploy mdBook to GitHub Pages](https://github.com/anjn/iron-aie-programming-guide/actions/workflows/deploy.yml/badge.svg)](https://github.com/anjn/iron-aie-programming-guide/actions/workflows/deploy.yml)

[Xilinx/AMD mlir-aie](https://github.com/Xilinx/mlir-aie) プロジェクトの[Programming Guide](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide)を日本語に翻訳した非公式ガイドです。

## オンライン版

このガイドは GitHub Pages で公開されています：
- **URL**: https://anjn.github.io/iron-aie-programming-guide/

## ローカルでのビルド

### 必要なツール

- [mdBook](https://rust-lang.github.io/mdBook/) v0.4.0以降

### インストール

```bash
# mdBookのインストール（Rustが必要）
cargo install mdbook

# またはバイナリを直接ダウンロード
# https://github.com/rust-lang/mdBook/releases
```

### ビルドと表示

```bash
# ビルド
mdbook build

# ローカルサーバーで表示（http://localhost:3000）
mdbook serve

# ブラウザを自動的に開く
mdbook serve --open
```

## プロジェクト構成

```
iron-programming-guide/
├── book.toml              # mdBookの設定
├── src/                   # ソースファイル
│   ├── SUMMARY.md        # 目次
│   ├── README.md         # はじめに
│   ├── quick_reference.md
│   ├── section-0/        # Section 0: 環境セットアップ
│   ├── section-1/        # Section 1: 基本的な構成要素
│   ├── section-2/        # Section 2: データ移動
│   ├── section-3/        # Section 3: はじめてのプログラム
│   ├── section-4/        # Section 4: 性能測定とベクトルプログラミング
│   ├── section-5/        # Section 5: ベクトル設計の例
│   ├── section-6/        # Section 6: 大規模設計の例
│   ├── mini_tutorial/    # ミニチュートリアル
│   └── theme/           # カスタムCSS
└── .github/workflows/   # GitHub Actions
```

## 貢献について

この翻訳は進行中のプロジェクトです。翻訳の改善や追加に貢献したい方は、以下の方法でご協力ください：

1. このリポジトリをフォーク
2. 新しいブランチを作成 (`git checkout -b feature/translation-improvement`)
3. 変更をコミット (`git commit -am '翻訳の改善: セクション3'`)
4. ブランチにプッシュ (`git push origin feature/translation-improvement`)
5. プルリクエストを作成

### 翻訳ガイドライン

- 技術用語は適切な日本語訳を使用するか、必要に応じて英語のままにする
- 元の文書の構造とフォーマットを維持する
- コードサンプルは変更しない
- 理解を助けるための注釈は歓迎

## ライセンス

この翻訳は非公式のものです。元のドキュメントは [Apache License v2.0 with LLVM Exceptions](https://llvm.org/LICENSE.txt) の下でライセンスされています。

## 元のプロジェクト

- **公式リポジトリ**: https://github.com/Xilinx/mlir-aie
- **公式プログラミングガイド**: https://github.com/Xilinx/mlir-aie/tree/main/programming_guide
- **公式ドキュメント**: https://xilinx.github.io/mlir-aie/

## 免責事項

これは非公式の翻訳であり、AMD/Xilinxによる公式なものではありません。正確な情報については、常に[公式の英語版ドキュメント](https://github.com/Xilinx/mlir-aie/tree/main/programming_guide)を参照してください。

翻訳の誤りや不正確な情報を見つけた場合は、Issueを作成するか、プルリクエストを送信してください。
