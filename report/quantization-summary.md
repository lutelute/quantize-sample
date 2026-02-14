# LLM量子化 — 簡潔まとめ

## 1. 量子化とは

モデルの重みの数値精度を落とし、メモリ・計算コストを削減する技術。

$$\text{VRAM} = \frac{N \times b}{8} \text{ [bytes]}$$

| 形式 | ビット幅 | 値の数 | 7Bモデル VRAM | 品質 |
|------|---------|--------|---------------|------|
| FP32 | 32 | ~42億 | 28.0 GB | 基準 |
| FP16 | 16 | ~65,536 | 14.0 GB | ≈同等 |
| INT8 | 8 | 256 | 7.0 GB | 微減 |
| INT4 | 4 | 16 | 3.5 GB | 顕著に減 |

![VRAM比較](figures/fig1_vram_comparison.png)

## 2. 量子化パイプライン

```mermaid
graph LR
    A["FP32 重み<br/>x = 0.73"] -->|"s = α / q_max"| B["Scale計算"]
    B -->|"q = round(x/s)"| C["Round & Clamp"]
    C -->|"x̂ = q · s"| D["逆量子化<br/>(近似値)"]
    style A fill:#1a3060,stroke:#4da6ff,color:#e8ecf2
    style B fill:#0a2018,stroke:#44cc66,color:#e8ecf2
    style C fill:#1a1508,stroke:#ffaa33,color:#e8ecf2
    style D fill:#1a1030,stroke:#b388ff,color:#e8ecf2
```

量子化誤差: $\varepsilon = |x - \hat{x}| \le s/2$

![パイプライン図](figures/fig4_pipeline.png)

## 3. 3つの量子化方式

```mermaid
graph TD
    Q["量子化方式の選択"] --> S["対称量子化"]
    Q --> A["非対称量子化"]
    Q --> B["ブロック量子化"]
    S --> S1["s = α / q_max<br/>x̂ = round(x/s) · s"]
    S --> S2["0が正確に保持<br/>ReLU後のスパーステンソル向き"]
    A --> A1["zero-point z を導入<br/>偏った分布に対応"]
    A --> A2["活性化値の量子化に有効"]
    B --> B1["ブロック単位でscale計算<br/>外れ値を局所化"]
    B --> B2["GPTQ / AWQ / GGUF<br/>実用手法の基盤"]
    style S fill:#0a2018,stroke:#44cc66,color:#e8ecf2
    style A fill:#1a1508,stroke:#ffaa33,color:#e8ecf2
    style B fill:#1a1030,stroke:#b388ff,color:#e8ecf2
```

| 方式 | 特徴 | 用途 |
|------|------|------|
| 対称 | $s = \alpha / q_{\max}$, zero-point不要 | 重み (0中心分布) |
| 非対称 | zero-point $z$ でオフセット補正 | 活性化値 (偏った分布) |
| ブロック | ブロック毎に独立scale, $b_{\text{eff}} = b + 16/g$ | 実用 (GPTQ, AWQ等) |

## 4. 重み分布と量子化誤差

重みは正規分布 $w \sim \mathcal{N}(0, \sigma^2)$ に近く、95%以上が $\pm 3\sigma$ 内に集中。
これが量子化が成立する根本的理由。

![重み分布](figures/fig2_weight_distribution.png)

**SQNR (信号対量子化雑音比)**:  1bitあたり約6 dB。

$$\text{SQNR} \approx 6.02 \, b + C \;\text{[dB]}$$

![誤差散布図](figures/fig3_error_scatter.png)

## 5. 実用手法の比較

```mermaid
graph LR
    subgraph PTQ["Post-Training Quantization"]
        GPTQ["GPTQ<br/>ヘッセ行列ベース<br/>列ごとに最適化"]
        AWQ["AWQ<br/>活性化ベースの<br/>重要度スケーリング"]
        GGUF["GGUF<br/>CPU推論向け<br/>k-quant混合精度"]
    end
    subgraph SM["Smoothing"]
        SQ["SmoothQuant<br/>活性化の外れ値を<br/>重みに移す"]
    end
    subgraph MX["Mixed-Precision"]
        LI8["LLM.int8()<br/>上位0.1%をFP16<br/>残りINT8"]
    end
    style GPTQ fill:#1a1030,stroke:#b388ff,color:#e8ecf2
    style AWQ fill:#0a2018,stroke:#44cc66,color:#e8ecf2
    style GGUF fill:#1a1508,stroke:#ffaa33,color:#e8ecf2
    style SQ fill:#10102a,stroke:#4da6ff,color:#e8ecf2
    style LI8 fill:#200a10,stroke:#ff5555,color:#e8ecf2
```

| 手法 | アプローチ | 特徴 |
|------|-----------|------|
| GPTQ | ヘッセ行列で誤差分散 | 高品質、GPU向け |
| AWQ | 活性化重要度でscale調整 | バランス良好 |
| GGUF | k-quant混合精度 | CPU推論、llama.cpp |
| SmoothQuant | 活性化→重みに外れ値移動 | W8A8で高速 |
| LLM.int8() | 混合精度 (FP16+INT8) | 外れ値チャネル保護 |

## 6. 外れ値問題

1個の外れ値 ($|\alpha_o| \gg |\alpha_n|$) がscaleを支配し、正常値の有効ビット幅が崩壊:

$$\text{有効bit} = b - 1 + \log_2\!\left(\frac{\alpha_n}{\alpha_o}\right)$$

$\alpha_o / \alpha_n = 100$ のとき、INT4の正常値は**実質情報量ゼロ**。

## 7. トレードオフの要点

```mermaid
quadrantChart
    title 品質 vs 効率のトレードオフ
    x-axis "低効率" --> "高効率"
    y-axis "低品質" --> "高品質"
    "FP32": [0.1, 0.95]
    "FP16": [0.3, 0.93]
    "INT8": [0.6, 0.85]
    "INT4 (AWQ)": [0.8, 0.75]
    "INT4 (naive)": [0.8, 0.55]
    "INT2": [0.95, 0.2]
```

**実用ライン**: INT4 (AWQ/GPTQ) がギリギリ実用可能。INT2以下は品質崩壊。

---

## 付録: インタラクティブツールのスクリーンショット

| タブ | スクリーンショット |
|------|-------------------|
| なぜ量子化？ | ![Tab0](figures/screenshot_tab0_why.png) |
| 概要と可視化 | ![Tab1](figures/screenshot_tab1_overview.png) |
| 計算プレイグラウンド | ![Tab2](figures/screenshot_tab2_playground.png) |
| 品質比較 | ![Tab3](figures/screenshot_tab3_compare.png) |
| 深掘り分析 | ![Tab4](figures/screenshot_tab4_deep.png) |
