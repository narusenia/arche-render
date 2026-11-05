# ArcheRender 研究方針と要件

> 状態: 合意済み（2026-11-06）
> 背景: [`rendering_research_themes.md`](../../rendering_research_themes.md)（MORT / DELTA / ALTO の研究仮説）

ArcheRender は、光輸送の計算量を減らす手法（MORT / DELTA / ALTO）を検証するための軽量レンダラーである。最終的には有効と確認できた手法を Verion（`narusenia/verion`）へ移植する。主目的は Verion の機能化だが、投稿の有無にかかわらず論文水準の厳密さで研究する。

## 1. 進め方

GPU レンダラーは最初から作らない。手順は次のとおり。

1. 先行研究を調査し、新規性マトリクスを作る。
2. 研究プロトコル（仮説・反証条件・判定閾値）を事前登録する。
3. CPU 小実験で仮説を検証する。
4. 仮説が判定ゲートを通過した場合に限り GPU 実装へ進む。
5. GPU 実装で再評価し、通過したものを Verion へ移植する。

MORT は先行研究調査の結果、新規性の余地が狭いと判断した（[`mort/survey.md`](mort/survey.md)）。そのため研究の主軸から外し、**機能化トラック**へ移す（2026-11-06）。ArcheRender で作り込んだうえで Verion へ移植する。

主軸の研究テーマは **DELTA** とし、主題を「多フレームにわたる差分推定の誤差の蓄積と、不確かさに基づくフル再計算との切替・結合」に絞る（2026-11-06、[`delta/protocol.md`](delta/protocol.md)）。DELTA の各要素は単独では既知だが、この点は RPI と Günther 2015 が未解決のまま残している（[`delta/survey.md`](delta/survey.md)）。

新テーマ探索で見つけた **A: 段差のある Toon 関数の閾値判定サンプリング**は、新規性のゲートを通過したが、第二候補として保留する（[`toon/survey.md`](toon/survey.md)）。ALTO も保留する。

| ID | 要件 |
|---|---|
| REQ-RES-001 | 事前登録の凍結は、先行研究調査と新規性マトリクスの完成後に行う。CPU 基盤と Oracle 実験の実装は調査と並行してよい |
| REQ-RES-002 | 調査の結果、新規性が残らない場合は、仮説を修正してから事前登録する |
| REQ-RES-003 | 判定閾値は実験前にプロトコルへ記載し、git tag で凍結する。凍結後に変更する場合は、理由を付けた改訂履歴をプロトコルに追記する |
| REQ-RES-004 | 機能化トラックのテーマ（MORT）は事前登録を要しない。ただし §2 の評価原則で効果を測り、プロトコルの判定ゲートを採用基準として使う |

## 2. 評価の原則

| ID | 要件 |
|---|---|
| REQ-EVAL-001 | 効率は `1 / (relMSE × コスト)` で測る。基準は PT + NEE + MIS とする |
| REQ-EVAL-002 | CPU 段のコストは計算量モデルで数える（交差レイ・可視性レイの本数と、L0 参照回数 × 重み係数の和）。CPU の実時間は参考値として併記するにとどめる。GPU 段では GPU-ms を正とする |
| REQ-EVAL-003 | 各条件を 16 seed 以上で実行し、95% 信頼区間を報告する |
| REQ-EVAL-004 | 不偏性を主張する推定器は、多 seed 平均と参照値の差を z 検定して確認する |
| REQ-EVAL-005 | 平均だけでなく、最悪シーンと失敗シーンも報告する。バイアス・分散・（動画の場合は）時間方向の蓄積誤差を分けて記録する |
| REQ-EVAL-006 | 新しい機構ごとに、その機構だけを無効化したアブレーションを行う |

### 正解画像

- 自前の CPU パストレーサーを Mitsuba 3 と照合し、FLIP / relMSE が閾値内に収まったら、その高 spp 結果を実験の正解とする（Verion ADR-0007 と同じ方針）。
- 照合の閾値は初回照合時に決め、プロトコルに記載する。

### シーン

自作の glTF だけを使う。外部シーンはライセンス確認が必要になるため使わない。

| 区分 | シーン |
|---|---|
| 判定対象（5） | ① Cornell 拡散 ② Cornell 小光源 ③ 半開きの扉（狭い開口・間接光主体） ④ 多重反射の室内（数千 tri） ⑤ GGX 光沢を含む Cornell |
| 失敗探査 | 鏡面、ガラス、コースティクス。結果は報告するが、閾値判定には含めない |

## 3. 実装方針

| ID | 要件 |
|---|---|
| REQ-IMPL-001 | 言語は Rust。Verion の crate（`verion-scene`、`verion-render-io` など）を Cargo の git 依存として `rev` 固定で参照する |
| REQ-IMPL-002 | Verion を同時に編集するときは、コミットしない `.cargo/config.toml` の `[patch]` でローカルの checkout に差し替える |
| REQ-IMPL-003 | CPU 段の crate は `arche-lab` の 1 つから始める（lib + 実験用 bin）。BVH・サンプリング・推定器・コストカウンタを含む。分割は GPU 段に入るときに行う |
| REQ-IMPL-004 | CPU 交差判定は自前の単純な BVH（中点分割）で実装し、外部の BVH ライブラリは使わない |
| REQ-IMPL-005 | CPU 段のコードは macOS と Windows の両方でビルドできる状態を保つ |
| REQ-IMPL-006 | GPU 段は Metal + Slang とし、`verion-gpu-metal` と `pathtrace.slang` を流用する。不足する API は Verion 側に追加する。wgpu などの別系統は作らない |
| REQ-IMPL-007 | 推定器のロジックは Slang に集約し、Metal 固有の機能に依存させない。Vulkan（Windows）へ移せる余地を残す。ホスト側の抽象層は Vulkan 着手時まで作らない |
| REQ-IMPL-008 | 依存は商用再配布が可能なものに限る。Verion の `deny.toml` を流用して `cargo deny` で検査する。GPL のベースライン実装（PPG・EARS・MARS の参照コードなど）はリンクせず、比較用の別バイナリとして動かす |

## 4. ツールと成果物の管理

- ツールは mise で管理する（Rust、Python 3.12、uv、Mitsuba 3、FLIP）。構成は Verion の `mise.toml` に合わせる。
- 統計処理と図の作成は、PEP 723 のインライン依存（numpy / scipy / matplotlib）を書いた Python スクリプトで行い、`uv run analysis/*.py` で実行する。Rust 側は `metrics.csv` を出力するだけにする。
- 実行結果は `results/<run-id>/` に置き、次の 2 つをコミットする。
  - `manifest.json`: ArcheRender の rev、Verion の rev、seed、機種、設定
  - `metrics.csv`
- EXR は `.gitignore` に入れ、manifest から再生成できるようにする。論文用の図だけを PNG でコミットする。
- 文書は日本語、コード・コメント・コミットメッセージは英語で書く。

## 5. 文書の配置

```
docs/research/
  README.md            本書（方針と要件）
  mort/protocol.md     MORT の研究プロトコル（事前登録）
  mort/survey.md       MORT の先行研究調査と新規性マトリクス
  delta/survey.md      DELTA の先行研究調査
  delta/protocol.md    DELTA の研究プロトコル（事前登録）
  toon/survey.md       新テーマ探索とテーマ A の先行研究（第二候補として保留）
```

## 6. 採用しなかったもの

- **Lean による形式検証**: MORT の不偏性は代数的な恒等式と Neumann 級数の収束条件だけで成り立つため、手書きの証明と統計検定で足りる。自明でない中心定理（分散上界の条件、低ランク近似の誤差伝播など）が出てきたら、改めて検討する。
- **Jupyter Notebook**: 再現性と差分レビューを優先し、スクリプトにした。
- **Git LFS による EXR 保存**: 再生成できるため保存しない。

## 7. 未決事項

- ライセンス: Verion（ADR-0004）と揃えて保留する。論文のコードを公開するときに決める。決まるまでは、第三者がこのコードを合法に利用できない。
- CI: GitHub Actions（macOS + Windows）で fmt / clippy / test を実行する案。まだ合意していない。
- 判定シーンの具体的な寸法・材質・光源パラメータ。
