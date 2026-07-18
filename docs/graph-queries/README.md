# sagebase_graph データマート 代表クエリ集

`sagebase-gcp.sagebase_graph` の graph-ready ノード/エッジテーブルと PROPERTY GRAPH `politics` を使った
**代表クエリ 5 種**を、**GQL 版とプレーン SQL 版の両方**で収録します。

> データセットは Analytics Hub の **`sagebase_graph_data`** リスティングとして公開されています。
> 購読手順は [../../README.md#購読手順bigquery-console](../../README.md#購読手順bigquery-console) を参照。
> 本ドキュメントの SQL 例はサンプル数値と実行しやすさを優先して `sagebase-gcp.sagebase_graph` を
> 直接参照する形で書いていますが、購読者が自分の linked dataset に対して発行する場合は
> `sagebase-gcp.sagebase_graph` → `<自分の project>.<linked dataset>` に読み替えてください。

## なぜ両形式を併記するか

BigQuery Graph（GQL）の実行には本来 Enterprise/Enterprise Plus エディションの予約スロットが要件とされます。
購読者の大半は GQL を実行できないため、**誰でも JOIN / 再帰 CTE で使えるプレーン SQL こそが主役**です。
GQL は「Enterprise 予約がある購読者向けの付加価値 + 技術ショーケース」と位置づけます。

各クエリで GQL 版と SQL 版は**同じ結果を返す**ことを実機で確認しています。

## クエリ一覧

| # | クエリ | 主役エッジ | 想定購読者 |
|---|--------|-----------|-----------|
| [01](01-voting-similarity.md) | 投票類似度ネットワーク | `VOTED_ON`（個人）/ `GROUP_VOTED_ON`（会派） | データジャーナリスト・政治学研究者 |
| [02](02-co-submission.md) | 共同提出ネットワーク | `SUBMITTED`（個人）/ `MEMBER_OF`（同型実例） | 政治学研究者・市民オンブズマン |
| [03](03-defection-analysis.md) | 会派方針との乖離（造反）分析 | `VOTED_ON.is_defection` × `MEMBER_OF` | 政治記者・選挙アナリスト |
| [04](04-deliberation-path.md) | 議案の審議経路追跡 | `DISCUSSED_IN` / `DELIBERATED_BY` | 立法過程研究者・政策ウォッチャー |
| [05](05-as-of-snapshot.md) | as-of スナップショット | `MEMBER_OF` + `GROUP_VOTED_ON` | 時系列分析者・歴史研究者 |
| [07](07-vote-events.md) | VoteEvent 経由の採決追跡 | `CAST_VOTE` / `GROUP_POSITION` / `DECIDES` / `HELD_AT` | 立法過程研究者・政治学者 |
| [08](08-presided-and-attended.md) | 議長経験と出席率（Phase 1 は国会のみ） | `PRESIDED` / `ATTENDED` | 政治学研究者・報道機関 |
| [可視化](visualization.md) | BigQuery Studio / NetworkX 可視化手順 | — | 全購読者 |

## 設計原則（クエリを書くときの約束）

| 原則 | このクエリ集での現れ方 |
|------|----------------------|
| **キーは `sagebase_id`（UUID/STRING）** | 全クエリの結合キーは `sagebase_id`。INT64 内部 id・FK は graph 層に存在しない |
| **時間軸の二分** | 期間型エッジ（`MEMBER_OF` 等）は `BETWEEN start_date AND COALESCE(end_date, '9999-12-31')`、時点型エッジ（`VOTED_ON.voted_date` 等）は等値/範囲で書く |
| **名寄せ品質の明示** | `SPOKE_IN` を使うクエリは `min_matching_confidence` / `all_verified` を WHERE で必ず使う（[04](04-deliberation-path.md) 参照） |
| **会派文脈の保持** | 造反分析（[03](03-defection-analysis.md)）は `VOTED_ON.parliamentary_group_sid` と `MEMBER_OF` の期間整合で会派を確定する |

## ⚠️ データカバレッジ注記（必読）

代表クエリ 5 種が依存する個人レベルのエッジは、**投入・名寄せが段階的に進行中**です。
「反映済み・部分反映・未反映（0 行）」を透明に区別して公開しています。値が入り次第、購読者側は追加作業なしで同じクエリが結果を返します。

| 影響エッジ | 状態 |
|-----------|------|
| `edges_voted_on`（個人投票） | 会派採決の個人展開と参議院の記名投票を投入済み。主に国会スコープ、地方議会は今後拡充 |
| `edges_voted_on.is_defection`（造反フラグ） | 現時点で全行 NULL。会派方針の確定処理が未反映（値が入り次第の後続リリースで反映） |
| `edges_submitted` / `edges_group_submitted` / `edges_conference_submitted`（提出系） | 現時点で 0 行。提出者マスタは投入済みだが、政治家/会派/会議体との紐付け（名寄せ）を本番 BQ に反映中 |
| `edges_discussed_in` / `edges_deliberated_by` | 段階拡充中。源泉 `proposal_deliberations` 投入は進行中 |
| `edges_affiliated_with`（立候補時政党） | 反映済（国会レベル） |
| `edges_spoke_in`（発言集約） | speaker→politician 名寄せ由来のためカバレッジは低め。各行に `min_matching_confidence` / `all_verified` を持つ |

このため各クエリ doc は **(a) 設計通りの正準クエリ**（源泉が埋まれば nightly で自動的に結果を返す再利用テンプレート）と、
**(b) 最も近い実データエッジでの同型クエリ**（`GROUP_VOTED_ON` / `MEMBER_OF` 等で実出力と性能を確認）の
両方を収録しています。正準クエリの実機実行ログ（0 行）も「SQL/GQL が構文上正しく走る」証跡として記録しています。

## 性能の要点（全クエリ共通・実測）

- **課金バイトは GQL が常に 250 MB 固定**（要素テーブル 25 本 × オンデマンド最小課金 10 MB の下限）。
  一方プレーン SQL は実際に触れたテーブルのみ課金され **20〜50 MB** で済みます。**コスト面でも SQL が有利**です。
- 単純な集約・2 ホップ射影は SQL/GQL とも **数十〜数百 slot-ms**（無視できる）。
- パス探索系（量化パス・`ANY SHORTEST`）は slot 集約的。購読者向けには SQL（JOIN/再帰 CTE）を推奨し、GQL パス探索は付加価値用途に留めるのが安全です。

## 実行方法

```bash
# プレーン SQL / GQL とも同じコマンド（GQL は本文が GRAPH ... で始まる）
bq query --use_legacy_sql=false --use_cache=false \
  --project_id=<自分のプロジェクト> --location=asia-northeast1 < query.sql
```

- ロケーションは `asia-northeast1` 必須（PROPERTY GRAPH の要素テーブルがある dataset のロケーション）。
- GQL の `RETURN` は `AS` 必須。`ANY SHORTEST` はパス変数束縛（`MATCH p = ANY SHORTEST ...`）が必要。
- 購読者は `sagebase-gcp.sagebase_graph` を自分の linked dataset 参照に読み替えてください。

## 関連ドキュメント

- 購読者向け CREATE PROPERTY GRAPH DDL 配布:
  [`06-subscriber-property-graph-ddl.md`](06-subscriber-property-graph-ddl.md)
  — 購読後に自 project で GQL を使うための DDL + CTAS フォールバック手順
- 可視化手順（BigQuery Studio / NetworkX）: [`visualization.md`](visualization.md)
