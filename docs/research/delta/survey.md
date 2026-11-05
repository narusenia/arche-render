# DELTA 先行研究調査

> 状態: 第 1 版（2026-11-06）。RPI 2024、Günther & Grosch 2015、HiPR 2026 は本文を確認した。その他は §5 を参照。
> 結論: **単独で新規と言える要素はない。** 主軸のテーマには採らず、第二候補として保留する（[`../README.md`](../README.md) §1）。

## 1. 先行研究の要点

| 論文 | 書誌 | 何をしたか | DELTA との重なり |
|---|---|---|---|
| Residual Path Integrals（RPI） | Xu, Li, Georgiev, Hedstrom, Ramamoorthi. CGF 43(4), EGSR 2024（Best Paper）. doi:10.1111/cgf.15152 | f_d = f2 − f1(T(p))·\|T′\| を、動的物体と ghost 物体から始める 6 技法の MIS で推定する。剛体移動で 2〜5 倍、材質編集で 4〜16 倍（Correlated PT 比） | 差分推定、共通サンプル空間、移動物体からのサンプリングの中核。CPU 実装（Embree） |
| Consistent Scene Editing by Progressive Difference Images | Günther, Grosch. CGF 34(4), EGSR 2015. doi:10.1111/cgf.12677 | SPPM で旧シーンと新シーンを同じ乱数種で交互に描き、差分を推定する。画素ごとのマスクで補正とリセットを切り替える | 編集時の差分推定と、フル再計算へのフォールバック（ヒューリスティック）。OptiX 実装 |
| HiPR | Padilla, Tate, Ram, Yuksel. SIGGRAPH 2026 Real-Time Live!. arXiv 2606.26612 | 変更要素から光路の依存をたどり、タイルの再描画に GPU で優先度を付ける | 変更イベントから影響光路を索引する部分。Vulkan 実装。再計算なので差分推定ではない |
| Selective Photon Tracing | Dmitriev, Brabec, Myszkowski, Seidel. EGWR 2002. doi:10.2312/EGWR/EGWR02/025-036 | pilot photon と Halton 列の周期性で、影響を受けた photon 経路を見つける | 影響光路の検出 |
| Temporal Gradient-Domain PT | Manzi et al. TOG 35(6) 2016. doi:10.1145/2980179.2980256 | 時間方向の有限差分を Poisson 再構成する | 差分と shift。再構成部は有偏 |
| Image-space Control Variates | Rousselle, Jarosz, Novák. TOG 35(6) 2016. doi:10.1145/2980179.2982443 | 画像空間の制御変量による再描画。材質編集に対応 | DCC 編集 |
| GRIS / ReSTIR PT、Area ReSTIR | Lin et al. TOG 2022. doi:10.1145/3528223.3530158 ／ Zhang et al. TOG 43(4) 2024 | 時空間での経路再利用（hybrid shift） | 実時間の再利用。差分推定ではない |
| Prioritized Re-Rendering | Ulschmid et al. GRAPP 2024 / SN Comp. Sci. 2025. doi:10.1007/s42979-025-03863-z | 編集時に重要領域から再描画する | DCC 編集 |
| 微分レンダリングの境界サンプリング | Li 2018、Zhang 2020、Bangaru 2020、Xu et al. 2024 | 無限小の変化に対する境界項 | 有限変位にはそのまま使えない（§2） |

## 2. 数式上の注意

物体が有限の距離だけ動く場合、可視性の変化は Dirac 型の境界項にならない。物体が通過した領域（ghost と dynamic の差）の上の面積積分になる。RPI の ghost 起点サンプリングは、まさにこの領域を狙っている。edge sampling 系の手法は無限小の変化を対象にしているので、「境界指向サンプリング」は有限変位に合わせて定式化し直す必要がある。

## 3. 新規性の評価

- 差分推定は RPI、編集時の差分とリセットは Günther 2015、影響光路の検出は Dmitriev 2002 と HiPR 2026 が先行している。
- **DCC の look-dev 編集向けは混んでいる**（Günther、Rousselle、RPI の材質編集、Ulschmid、HiPR）。
- **アニメーション向けのほうが空いている。** RPI 自身が未解決としている点が残る。
  1. GPU 化。遅延とメモリの扱いも未設計
  2. 多フレームにわたる誤差の蓄積の解析と、不確かさに基づくフル更新への切替（Günther はヒューリスティックのみ）
  3. 変形する物体、多数の動的物体
  4. 光源とカメラの移動（RPI は対象外と明言）
  5. 差分の被積分関数に比例するサンプリング
- 実時間ハイブリッド向けは ReSTIR と DDGI の系統が強く、不偏な差分推定で入るのは難しい。
- **再開するなら**: 2（誤差蓄積と切替）を単独のテーマとして切り出すのが最も個人研究向き。

## 4. ベースラインと公開実装

| ベースライン | 実装 | ライセンス |
|---|---|---|
| フレームごとの PT | Mitsuba 3 / Falcor | BSD-3 系 |
| Correlated PT（同一乱数種の差分） | 自前で実装 | — |
| RPI | 公開コードは見つからない。再実装が必要 | — |
| Temporal GPT | mkettune/temporal-gpt | GPL-3.0（リンクしない） |
| ReSTIR PT | DQLin/ReSTIR_PT | BSD-3 |
| DDGI | NVIDIAGameWorks/RTXGI-DDGI | NVIDIA 独自 |
| 境界サンプリング | BachiLi/redner、uci-rendering/psdr-cuda | MIT、BSD-3 |

## 5. 未確認

- Temporal GPT 2016、Rousselle 2016、ReSTIR PT、Selective Photon Tracing の本文（Selective Photon Tracing の正負 photon 補正は Günther の記述に頼っている）。
- Ulschmid 2024/25 と MCMC 境界サンプリング 2024 は抄録のみ。
- Real-Time LoD ReSTIR 2026 の著者。
- 2025〜26 年の EGSR・SIGGRAPH の目次は人手で確認していない。再開するときに確認する。
