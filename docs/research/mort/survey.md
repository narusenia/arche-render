# MORT 先行研究調査と新規性マトリクス

> 状態: 第 1 版（2026-11-06）。書誌は Web 検索と著者ページで確認した。本文まで読めていない論文は §5 に挙げる。
> 結論を先に書く。**MORT の構成要素は、どれも単独では既知である。** 新規性が残りうるのは、多階層の制御変量 L0 に対して、レベルごとの残差分散と GPU コストからレベル間の予算をオンラインで配分する点に限られる。

## 1. 定式化の位置づけ（自己点検）

protocol の推定器を頂点 x で展開すると、次のようになる。

$$
\hat L(x) = L_0(x) + \bigl[E(x) + \widehat{\mathcal K L_0}(x) - L_0(x)\bigr] + \widehat{\mathcal K\Delta L}(x)
= E(x) + \widehat{\mathcal K L_0}(x) + \widehat{\mathcal K (L - L_0)}(x).
$$

L0(x) は打ち消し合う。したがって「放射源を r=E+KL0−L0 に置き換えて ΔL を輸送する」形は、**各頂点で入射放射輝度 L0 を制御変量にする 2 レベル推定器（NCV・NIRC と同じ型）を、すべての頂点に再帰的に適用したもの**と代数的に等しい。違いが出るのは、L0(x) を打ち消さずに使う有偏の打ち切り版だけである。

→ 残差輸送の定式化そのものは新規性として主張しない。protocol §2 の「主仮説」は「既知の型の推定器を多階層化したもの」と位置づけ直す。

## 2. 新規性マトリクス

◎=同等の要素あり、○=部分的、−=なし

| 研究 | L0 を CV に使う | 全頂点で残差 | 多階層 L0 | コスト考慮配分 | 不偏 | GPU 実時間 |
|---|---|---|---|---|---|---|
| Lafortune & Willems 1995（5D tree） | ◎ | ?（要本文） | ○ 適応細分 | − | ◎ | − |
| Keller 2001（Hierarchical MC Image Synthesis） | ○ | − | ◎ 画素積分の階層 CV | ? | ◎ | − |
| NCV（Müller+ 2020） | ◎ | ? | − | − | ◎ | ○ |
| Crespo+ 2021（Primary-space adaptive CV） | ◎ | ○ | ○ 適応細分 | ? | ◎ | − |
| Salaün+ 2022（Regression-based MC） | ○ | − | − | − | ◎ | − |
| **NIRC（Dereviannykh+ EG 2025）** | ◎ | ○ 最初の 3 頂点 | ○ 2 レベルのみ（多レベルは future work） | − 手動調整（分散を考慮した配分は future work） | ◎ | ◎ |
| Lee+ 2025（MLMC PT, Visual Computer） | − | − | ○ レベル＝経路の深さ | ◎ バンディット | ◎ | ? |
| Xu & Wang EGSR 2025（Adaptive multiple CV） | ○ 直接照明のみ | − | ○ 光源の階層 | ○ CV 数 | ◎ | ? |
| Neural Radiosity（Hadadan+ 2021） | − r のノルムを学習目的に使う | − | − | − | − | − |
| RCPG（Radiance Cascades PG） | − guiding に使う | − | ◎ cascade 階層 | − | ◎ | ◎ |
| Illumination-Aware Subdivision（EGSR 2026） | − guiding に使う | − | ◎ | ○ 不確かさで配分 | ◎ | − |
| EARS（Rath+ 2022）/ MARS（Meyer+ 2024） | − | − | − | ◎ 効率 = 1/(分散×コスト) の最大化 | ◎ | − |
| Hierarchical / Wavelet radiosity | − | − | ◎ 演算子の階層 | ○ 誤差で精度配分 | − | − |
| **MORT（候補）** | ◎ | ◎ | ◎ 空間・方向・周波数 | ◎ レベル別の分散・GPU-ms | ◎ | 目標 |

## 3. 新規性の評価

- **既知**: L0 を制御変量にした不偏推定（Lafortune 1995、NCV、NIRC）、多階層の放射輝度表現（cascades、多解像度ハッシュ、適応木）、効率最大化による配分（EARS / MARS、Giles の MLMC 配分則）。
- **残りうる差分**: 制御変量の**階層軸**に対する効率最大化の配分。EARS が経路の分岐数で行っている最適化を、L0 のレベル数・レベルごとのサンプル数に移す。最も近い NIRC は 2 レベル・手動配分で、多レベル化も分散を考慮した配分も future work と書いている。
- **リスク**:
  - 査読者に「NIRC の自然な拡張」と見なされやすい。
  - NIRC の著者グループ（KIT Dachsbacher 研）が先に出す可能性がある。
- **主張の絞り込み**: 「NIRC の 2 レベル・手動配分を、L レベル・コスト考慮の自動配分へ一般化する。L0 はニューラルネットワークを使わない軽量な疎グリッドで実現し、同じ GPU-ms で NIRC より低い誤差を出す」。

## 4. ベースライン（protocol §3 に反映）

| 段階 | ベースライン | 理由 | 実装 |
|---|---|---|---|
| S1〜 | PT + NEE + MIS | 効率比の基準 | 自前 |
| S2〜 | 単一レベル CV（NCV 型、同じグリッド L0） | 「多階層ではない」既知手法 | 自前（MORT の L=1） |
| S2〜 | NIRC 型 2 レベル（手動配分） | 最も近い先行研究 | 自前で再現（公開コードの有無は未確認） |
| S2〜 | PPG（SD-tree）と MORT+PPG | guiding と補完的かを見る | Intel Open PGL（Apache-2.0）を第一候補とする |
| S2〜 | PT + EARS | 配分機構の寄与を切り分ける | 自前で再実装。参照実装は GPL-3.0 なのでリンクしない |
| S3 | Lee+ 2025（深さ方向 MLMC 配分） | 配分を新規性として主張するため | 本文確認後に判断 |
| 参考 | NRC（有偏） | 有偏キャッシュの参照点 | Aerion 向けの副次実験で使う |

GPL-3.0 のコード（Tom94/practical-path-guiding、iRath96/ears、woshicado/mars）は ArcheRender にリンクしない。参照として動かすときは、別バイナリで比較実験を回す（REQ-IMPL-008）。

## 5. 未確認（凍結前に解消する）

1. **NIRC 本文**: 残差の推定がレベルを再帰するか、§7 の future work の正確な記述、コードとライセンスの有無。差分の主張を左右する最重要項目。
2. Lafortune 1995: 5D tree の CV を全バウンスで再帰的に適用しているか。
3. Keller 2001: レベル間の配分最適化があるか。
4. Lee+ 2025（Visual Computer）: 共著者と本文。
5. NCV: 残差をどの頂点まで扱うか。
6. RCPG: 出版年（2026 か 2027 か）、cascade の角度方向の階層。
7. DOI 未確認: Lafortune の ambient 論文、Dodik 2022、Wavelet radiosity 1993、Heinrich 2001。
8. 2025〜2026 年に同じ組合せの論文がないか。検索が網羅的であるとは保証できない。凍結の直前にもう一度検索する。

## 6. 参考文献（確認済み）

- Lafortune, Willems. A 5D Tree to Reduce the Variance of Monte Carlo Ray Tracing. EGWR 1995. doi:10.1007/978-3-7091-9430-0_2
- Keller. Hierarchical Monte Carlo Image Synthesis. Math. Comput. Simul. 55, 2001. doi:10.1016/S0378-4754(00)00248-2
- Giles. Multilevel Monte Carlo Path Simulation. Oper. Res. 56(3), 2008. doi:10.1287/opre.1070.0496
- Rousselle, Jarosz, Novák. Image-space Control Variates for Rendering. TOG 35(6), 2016. doi:10.1145/2980179.2982443
- Müller, Gross, Novák. Practical Path Guiding for Efficient Light-Transport Simulation. CGF 36(4), 2017. doi:10.1111/cgf.13227
- Müller, Rousselle, Novák, Keller. Neural Control Variates. TOG 39(6), 2020. doi:10.1145/3414685.3417804
- Rath et al. Variance-Aware Path Guiding. TOG 39(4), 2020. doi:10.1145/3386569.3392441
- Crespo, Jarabo, Muñoz. Primary-Space Adaptive Control Variates using Piecewise-Polynomial Approximations. TOG 40(3), 2021. doi:10.1145/3450627
- Müller, Rousselle, Novák, Keller. Real-time Neural Radiance Caching for Path Tracing. TOG 40(4), 2021. doi:10.1145/3450626.3459812
- Hadadan, Chen, Zwicker. Neural Radiosity. TOG 40(6), 2021. doi:10.1145/3478513.3480569
- Salaün et al. Regression-based Monte Carlo Integration. TOG 41(4), 2022. doi:10.1145/3528223.3530095
- Rath et al. EARS: Efficiency-Aware Russian Roulette and Splitting. TOG 41(4), 2022. doi:10.1145/3528223.3530168
- Meyer, Rath, Yazici, Slusallek. MARS. SIGGRAPH Asia 2024 Conf. doi:10.1145/3680528.3687636
- Dereviannykh, Klepikov, Hanika, Dachsbacher. Neural Two-Level Monte Carlo Real-Time Rendering. CGF 44(2), 2025. doi:10.1111/cgf.70050
- Xu, Wang. Adaptive Multiple Control Variates for Many-Light Rendering. EGSR 2025. doi:10.2312/sr.20251184
- Lee et al. Multilevel Monte Carlo for Asymptotically Efficient Path Tracing. The Visual Computer 41(9), 2025. doi:10.1007/s00371-025-03935-4
- Hanrahan, Salzman, Aupperle. A Rapid Hierarchical Radiosity Algorithm. SIGGRAPH 1991. doi:10.1145/127719.122740
- Zheng, Peters, Herholz, Manzi, Eisemann. Illumination-Aware Spatial Subdivision for Path Guiding. EGSR 2026.
- Ikkala, Jääskeläinen, Mäkitalo. RCPG: Real-Time Product Path Guiding Using Radiance Cascades. TOG. doi:10.1145/3840291
