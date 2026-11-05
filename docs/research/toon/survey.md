# 新テーマ探索と主軸テーマ A（Toon 閾値判定サンプリング）の先行研究

> 状態: 第 1 版（2026-11-06）。West 2024 と Tong 2025 は本文を確認した。
> 主軸テーマとして A を採用した（[`../README.md`](../README.md) §1）。ただし West, Mukherjee, Yue（SIGGRAPH 2026）の本文確認を着手前のゲートにする。

## 1. 探索の方針

大手研究所が詰めている分散削減ではなく、Verion の核である Art-Directable Physical Rendering（PBR + Stylization）に固有の問題を探した。

## 2. 既出だった種: スタイル化 = 期待値の非線形関数

- **West & Mukherjee, "Stylized Rendering as a Function of Expectation", TOG 43(4) 2024**, doi:10.1145/3658161
  - スタイル化を再帰的な f(E[·]) として定式化し、不偏と有偏の推定ツールキットを示した。
  - Limitations に未解決点として挙げているもの:
    1. **不連続関数（ステップ関数、セル塗り）について、収束する不偏推定**
    2. stylized path integral
    3. 畳み込み系のスタイル
    4. サンプル再利用
- **Tong & Hachisuka, "Practical Stylized Nonlinear Monte Carlo Rendering", SIGGRAPH 2025 Conf.**, doi:10.1145/3721238.3730686
  - NL-PF と NL-NRC で未解決点 4（サンプル再利用）を解決した。コードは公開されているが CUDA が必須。
  - 不連続については「放射輝度がちょうど不連続点に乗る面はない」と仮定して回避している。Toon の収束が遅いことも本文で認めている。
- **West, Mukherjee, Yue, "Lifting Lines and Tone: Image-space Stylization in Path-space"（SIGGRAPH 2026）**: 題名と著者は確認したが、本文は未読。未解決点 3 を同じグループが埋めにいっていると読める。

## 3. 候補と評価

| 候補 | 内容 | 新規性の余地 | 判断 |
|---|---|---|---|
| **A** | 段差のある Toon 関数 g の入れ子期待値 E_pix[g(E_light[L])] を、閾値を意識した適応サンプリングで推定する | 中〜高 | **主軸として採用** |
| B | スタイルの重み w→0 で Reference に収束する推定器 | 低（West 2024 でほぼ説明がつく） | A のベースラインと Reference 一致テスト（REQ-STY-007）に吸収する |
| C | スタイル化を考慮したデノイズ。閾値を条件に入れた事後補正 | 中 | A の拡張として第二段階で扱う |
| D | アートディレクション編集（light / shadow linking）と MIS・ReSTIR の整合 | 論文としては中〜低 | 研究ではなく実装項目として持つ（REQ-DCC） |
| E | スタイルパラメータを条件に入れた非線形放射輝度キャッシュ | 中 | 工数が大きく、CUDA 前提の先行研究からの移植も重い。保留 |
| — | LPE・成分分解とデノイズ | 低 | 混んでいる（Zhang+ 2021 Disney、Munkberg & Hasselgren 2020 など） |

## 4. 候補 A の詳細

### 問題

一次ヒットのセル塗りで求めたい値は、画素フットプリント上の外側の期待値の中に、光ごとの寄与の内側の期待値を段差関数 g に通したものが入った形 E_pix[g(E_light[L])] である。

- Verion の現行方式（ADR-0008）は AOV を画素で平均してから g を掛けるので、g(E_pix E_light) になる。幾何やテクスチャの境界で段がにじんだり、誤ったりする。
- サンプルごとに g を掛けると、段の境界がぼける（West 2024 の Fig.5）。
- g の両側の値はわかっているので、実際に必要なのは「E[L] が閾値 τ のどちら側にあるか」の判定である。これは thresholding bandit 問題に当たる。

### 仮説

同じ時間で比べると、閾値までの距離 |μ−τ|/σ に基づく配分（APT 型）は、分散ベースの適応サンプリング（DAAS 型）と一様配分よりも、段の境界での誤分類率と時間方向のちらつきを 2 倍以上減らす。

### 関連研究

- West 2024: 不連続な g を future work として残している。
- Tong 2025: 不連続を仮定で回避している。
- Locatelli et al., "An optimal algorithm for the Thresholding Bandit Problem", ICML 2016（arXiv:1605.08671）: 統計側の道具。描画への適用例は見つからなかった（検索 2 回の範囲）。
- Firmino et al., "Denoising-Aware Adaptive Sampling", SIGGRAPH 2023（doi:10.1145/3588432.3591537）: 非線形な出力の分散で配分するが、閾値判定ではない。
- Rainforth et al., "On Nesting Monte Carlo Estimators", ICML 2018: 外側がステップ関数の場合の収束率は未解決（Tong 経由で確認）。

### 新規性のリスク

- West 研が 2026 年の論文で不連続な g に触れている可能性を否定できない。
- 理論面で「既知の bandit を当てはめただけ」と言われやすい。主張するには、画素フットプリントの外側積分との組み合わせと、時間方向の安定性まで扱う必要がある。

### Verion との結びつき

- REQ-STY-002 / 007、ADR-0008、REQ-QA-003（ちらつき）、`docs/drafts/adaptive_sampling_scheduler_plan.md`。
- draft §8.6 の「Toon 境界では Adaptive Sampling を増やす」を理論の裏付けのある形にする。
- 未決事項の「HDRI 下の Toon の段」も、同じ枠組みの検証シーンにできる。

### 最小プロトタイプと撤退条件

- **最初の実験**: CPU で、球 1 個と面光源 1 個（ソフトシャドウ）、3 段の Ramp。内側に 10⁶ サンプルかけた参照を作る。次の 3 方式で誤分類率、FLIP、連番のちらつきを測り、現行方式の誤差が実際に目に見えるかを確かめる。
  1. 画素平均の AOV に g を掛ける（現行）
  2. サンプルごとに g を掛ける
  3. 入れ子推定
- **撤退条件**: 同じ時間で DAAS 型配分に対する境界誤差の改善が 1.3 倍未満。または誤分類が「境界ちょうど」の画素に集中し、どの配分でも差が出ない。
- **工数**: CPU 実験で 2〜3 週間程度。

## 5. 未確認

1. **West, Mukherjee, Yue（SIGGRAPH 2026）の本文**: 着手前のゲート。確認中。
2. West 2024 と Tong 2025 の被引用一覧。
3. 描画分野への thresholding bandit の適用例がないこと。検索 2 回の結果にすぎない。
4. Rainforth 2018 と Schmidt 2013 の本文。
