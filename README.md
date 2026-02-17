# 量子化ビジュアライザー

LLMの量子化（FP32 → FP16 → INT8 → INT4）をインタラクティブに学べるWebツール。

**[デモを見る](https://lutelute.github.io/quantize-sample/)**

## 概要

AIモデルの量子化プロセスを、スライダー操作やリアルタイムのグラフ更新を通じて直感的に理解できるビジュアライザーです。

### 主な機能

- **量子化パイプライン体験** — FP32の値がINT4になるまでの計算過程をステップごとに確認
- **画像の量子化デモ** — 色情報の段階的な削減を視覚的に体験
- **信号量子化と誤差** — SQNR（信号対量子化雑音比）の可視化
- **手法別比較** — GPTQ / AWQ / GGUF / SmoothQuant / LLM.int8() の特性比較
- **外れ値問題の解説** — 1個の外れ値がスケールを支配するメカニズム

## 技術構成

- 単一HTML（`index.html`）で完結、ビルド不要
- [Chart.js](https://www.chartjs.org/) — グラフ描画
- [KaTeX](https://katex.org/) — 数式レンダリング

## ローカルで実行

```bash
# 任意のHTTPサーバーで配信
npx serve .
# または
python3 -m http.server 8000
```

`index.html` をブラウザで直接開いても動作します。

## ファイル構成

```
index.html                          # メインアプリケーション
report/
  quantization-summary.md           # 量子化の技術サマリー（Markdown）
  figures/                          # 図表・スクリーンショット
.github/workflows/deploy.yml       # GitHub Pages 自動デプロイ
```

## レポート

`report/quantization-summary.md` に量子化の理論・手法・トレードオフを簡潔にまとめています。

| トピック | 内容 |
|---------|------|
| 量子化の基礎 | VRAM計算、ビット幅と値の数の関係 |
| パイプライン | Scale計算 → Round & Clamp → 逆量子化 |
| 3方式の比較 | 対称 / 非対称 / ブロック量子化 |
| 実用手法 | GPTQ, AWQ, GGUF, SmoothQuant, LLM.int8() |
| 外れ値問題 | 有効ビット幅の崩壊メカニズム |
