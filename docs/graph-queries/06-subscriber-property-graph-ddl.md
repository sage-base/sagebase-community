# 06. 購読者向け CREATE PROPERTY GRAPH DDL 配布

BigQuery Analytics Hub で `sagebase_graph_data` をサブスクライブした購読者が、**自分のプロジェクトの
dataset 上で** PROPERTY GRAPH `politics` を定義し、GQL グラフクエリを実行するための DDL と手順です。

> **なぜ DDL を配布するのか**: PROPERTY GRAPH オブジェクトそのものは BigQuery Sharing で共有できません。
> Sharing が共有するのは**テーブル（ノード/エッジ）**だけです。GQL を使いたい購読者は、共有された
> テーブルを参照する PROPERTY GRAPH を**自分の project に自分で定義する**必要があります。本ドキュメントは
> その DDL と適用手順を提供します。

> **前提（GQL は Enterprise 予約が要件）**: BigQuery Graph の GQL 実行には本来 Enterprise / Enterprise Plus
> エディションの予約スロットが要件とされます（公開元プロジェクトのオンデマンドでは動作していますが、購読者
> 環境での可否は保証しません）。GQL を実行できない購読者は、PROPERTY GRAPH を作らず**プレーン SQL
> （JOIN / 再帰 CTE）**だけで全データを利用できます（[README](README.md) と各クエリ doc 参照）。本ドキュメントは
> 「GQL も使いたい購読者向けの付加価値」です。

---

## 全体の流れ

```
1. sagebase_graph_data をサブスクライブ → 自 project に linked dataset 作成
2. (A) linked dataset を直接参照して CREATE PROPERTY GRAPH を発行
       ↓ 権限・参照制約で不可だった場合のフォールバック
   (B) linked dataset の各テーブルを CTAS で自 dataset に物理コピー → そのコピーを参照して
       CREATE PROPERTY GRAPH を発行
3. GQL / プレーン SQL でクエリ実行
```

このドキュメントで使うプレースホルダ:

| プレースホルダ | 意味 | 例 |
|----------------|------|-----|
| `<MY_PROJECT>` | 購読者自身の GCP プロジェクト ID | `my-analytics-proj` |
| `<LINKED_DATASET>` | サブスクライブ時に作成される linked dataset 名（購読者が命名） | `sagebase_graph_linked` |
| `<MY_DATASET>` | PROPERTY GRAPH と（フォールバック時の）物理コピーを置く購読者の**書込可能**な dataset | `sagebase_graph_work` |

> **ロケーション必須**: linked dataset・`<MY_DATASET>`・PROPERTY GRAPH のジョブは、共有元の要素テーブルと
> 同じ **`asia-northeast1`** に揃えてください（要素テーブルの dataset ロケーションに一致が必要）。

---

## ステップ 1: サブスクライブ → linked dataset 作成

BigQuery コンソール（または `bq` / API）で `sagebase_exchange` 内の `sagebase_graph_data` リスティングを
サブスクライブします。サブスクライブすると、自 project に**共有データセットへの read-only ポインタ**
である linked dataset（`<LINKED_DATASET>`）が作成されます。詳細な GUI 手順は
[../../README.md#購読手順bigquery-console](../../README.md#購読手順bigquery-console) を参照してください。

PROPERTY GRAPH を置くための**書込可能**な dataset（`<MY_DATASET>`）を `asia-northeast1` に用意する:

```bash
bq mk --location=asia-northeast1 --dataset <MY_PROJECT>:<MY_DATASET>
```

---

## ステップ 2-(A): linked dataset を直接参照して PROPERTY GRAPH を定義（第一選択）

購読者側の DDL は、**PROPERTY GRAPH 名と要素テーブルの dataset 修飾を自 project 向けに書き換える**
だけで、それ以外の構造（`REFERENCES` / `KEY` / `PROPERTIES`）は全て同一です:

1. **PROPERTY GRAPH 名**: `<MY_PROJECT>.<MY_DATASET>.politics`（購読者の書込可能 dataset に作る）
2. **要素テーブルの dataset 修飾**: `<LINKED_DATASET>.<table>`（linked dataset を参照する）

**変わらない点**:
- `REFERENCES` 側は **NODE TABLES で宣言したテーブル名のみ**（dataset 修飾しない）。
- ノード／エッジとも**明示 `KEY (...)` 必須**（PK 未定義の BQ テーブルでは省略不可）。
- 審議系（`DISCUSSED_IN` / `DELIBERATED_BY`）の `stage` は NULL を取りうるが、モデル側で決定的ハッシュ
  `sagebase_id` を物理化済みのため `KEY (sagebase_id)` を使う（NULL 複合キーのエッジは走査から無言で
  脱落するため）。
- `PROPERTIES` は公開列に限定（`PROPERTIES ALL COLUMNS` は使わない）。

以下が購読者向け完全 DDL（`<LINKED_DATASET>` と `<MY_PROJECT>.<MY_DATASET>` を自分の値に置換して発行）:

```sql
CREATE OR REPLACE PROPERTY GRAPH `<MY_PROJECT>.<MY_DATASET>.politics`
NODE TABLES (
  <LINKED_DATASET>.nodes_politician KEY (sagebase_id) LABEL Politician
    PROPERTIES (sagebase_id, name, furigana, kanji_name, prefecture, district, profile_page_url),
  <LINKED_DATASET>.nodes_political_party KEY (sagebase_id) LABEL PoliticalParty
    PROPERTIES (sagebase_id, name, members_list_url),
  <LINKED_DATASET>.nodes_parliamentary_group KEY (sagebase_id) LABEL ParliamentaryGroup
    PROPERTIES (sagebase_id, name, url, description, is_active, chamber, start_date, end_date),
  <LINKED_DATASET>.nodes_governing_body KEY (sagebase_id) LABEL GoverningBody
    PROPERTIES (sagebase_id, name, organization_code, organization_type, prefecture, type),
  <LINKED_DATASET>.nodes_conference KEY (sagebase_id) LABEL Conference
    PROPERTIES (sagebase_id, name, term, conference_type),
  <LINKED_DATASET>.nodes_meeting KEY (sagebase_id) LABEL Meeting
    PROPERTIES (sagebase_id, name, date, url),
  <LINKED_DATASET>.nodes_proposal KEY (sagebase_id) LABEL Proposal
    PROPERTIES (
      sagebase_id, title, proposal_category, proposal_type, session_number, proposal_number,
      deliberation_status, deliberation_result, submitted_date, voted_date, detail_url
    ),
  <LINKED_DATASET>.nodes_election KEY (sagebase_id) LABEL Election
    PROPERTIES (sagebase_id, name, term_number, election_date, election_type, office),
  <LINKED_DATASET>.nodes_vote_event KEY (sagebase_id) LABEL VoteEvent
    PROPERTIES (
      sagebase_id, voted_date, vote_type, result, stage, legislative_session,
      matching_confidence, is_llm_extracted
    )
)
EDGE TABLES (
  -- 所属系（期間型 is_current 付き）
  <LINKED_DATASET>.edges_member_of KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    LABEL MEMBER_OF PROPERTIES (sagebase_id, start_date, end_date, is_current, role),
  <LINKED_DATASET>.edges_member_of_conference KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_conference (sagebase_id)
    LABEL MEMBER_OF_CONFERENCE PROPERTIES (sagebase_id, start_date, end_date, is_current, role),
  -- 政党所属・立候補（時点型）
  <LINKED_DATASET>.edges_affiliated_with KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_political_party (sagebase_id)
    LABEL AFFILIATED_WITH PROPERTIES (sagebase_id, election_date),
  <LINKED_DATASET>.edges_ran_in KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_election (sagebase_id)
    LABEL RAN_IN PROPERTIES (sagebase_id, result, votes, rank),
  -- 発言集約（名寄せ品質付き）
  <LINKED_DATASET>.edges_spoke_in KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_meeting (sagebase_id)
    LABEL SPOKE_IN
    PROPERTIES (sagebase_id, meeting_date, speech_count, total_chars, min_matching_confidence, all_verified),
  -- 投票系
  <LINKED_DATASET>.edges_voted_on KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL VOTED_ON
    PROPERTIES (sagebase_id, approve, is_defection, parliamentary_group_sid, voted_date, source_type),
  <LINKED_DATASET>.edges_group_voted_on KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL GROUP_VOTED_ON
    PROPERTIES (sagebase_id, judgment, member_count, judge_type, group_name, voted_date),
  -- 提出系（submitter_type 全種: 政治家／会派／会議体）
  <LINKED_DATASET>.edges_submitted KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL SUBMITTED PROPERTIES (sagebase_id, is_representative, display_order, submitted_date),
  <LINKED_DATASET>.edges_group_submitted KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL GROUP_SUBMITTED PROPERTIES (sagebase_id, submitted_date),
  <LINKED_DATASET>.edges_conference_submitted KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_conference (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL CONFERENCE_SUBMITTED PROPERTIES (sagebase_id, submitted_date),
  -- 構造系（単一キーを持たないため source_sid + dest_sid〔+ stage〕を KEY とする）
  <LINKED_DATASET>.edges_part_of KEY (source_sid, dest_sid)
    SOURCE KEY (source_sid) REFERENCES nodes_conference (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_governing_body (sagebase_id)
    LABEL PART_OF,
  <LINKED_DATASET>.edges_belongs_to_gb KEY (source_sid, dest_sid)
    SOURCE KEY (source_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_governing_body (sagebase_id)
    LABEL BELONGS_TO_GB,
  <LINKED_DATASET>.edges_held_by KEY (source_sid, dest_sid)
    SOURCE KEY (source_sid) REFERENCES nodes_meeting (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_conference (sagebase_id)
    LABEL HELD_BY PROPERTIES (source_sid, dest_sid, meeting_date),
  <LINKED_DATASET>.edges_held_for KEY (source_sid, dest_sid)
    SOURCE KEY (source_sid) REFERENCES nodes_election (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_governing_body (sagebase_id)
    LABEL HELD_FOR PROPERTIES (source_sid, dest_sid, election_date),
  <LINKED_DATASET>.edges_composed_of KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_political_party (sagebase_id)
    LABEL COMPOSED_OF PROPERTIES (sagebase_id, start_date, end_date, is_current, is_primary),
  <LINKED_DATASET>.edges_discussed_in KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_proposal (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_meeting (sagebase_id)
    LABEL DISCUSSED_IN PROPERTIES (sagebase_id, stage),
  <LINKED_DATASET>.edges_deliberated_by KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_proposal (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_conference (sagebase_id)
    LABEL DELIBERATED_BY PROPERTIES (sagebase_id, stage),
  -- VoteEvent 系: 採決 reify。既存 edges_voted_on / edges_group_voted_on は購読者互換のため温存。
  <LINKED_DATASET>.edges_decides KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_vote_event (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_proposal (sagebase_id)
    LABEL DECIDES PROPERTIES (sagebase_id, stage, voted_date),
  <LINKED_DATASET>.edges_held_at KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_vote_event (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_meeting (sagebase_id)
    LABEL HELD_AT PROPERTIES (sagebase_id, voted_date),
  <LINKED_DATASET>.edges_cast_vote KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_vote_event (sagebase_id)
    LABEL CAST_VOTE
    PROPERTIES (sagebase_id, approve, is_defection, parliamentary_group_sid, attribution_rule, source_type),
  <LINKED_DATASET>.edges_group_position KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_parliamentary_group (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_vote_event (sagebase_id)
    LABEL GROUP_POSITION
    PROPERTIES (sagebase_id, judgment, member_count, judge_type, attribution_rule),
  -- 会議主宰・出席: 議事録冒頭 regex 抽出源。Phase 1 は国会のみ、地方議会は今後拡充。
  <LINKED_DATASET>.edges_presided KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_meeting (sagebase_id)
    LABEL PRESIDED
    PROPERTIES (
      sagebase_id, meeting_date, position, matching_confidence, source,
      is_manually_verified
    ),
  <LINKED_DATASET>.edges_attended KEY (sagebase_id)
    SOURCE KEY (source_sid) REFERENCES nodes_politician (sagebase_id)
    DESTINATION KEY (dest_sid) REFERENCES nodes_meeting (sagebase_id)
    LABEL ATTENDED
    PROPERTIES (
      sagebase_id, meeting_date, status, role_category, matching_confidence,
      source, is_manually_verified
    )
)
```

`CREATE OR REPLACE PROPERTY GRAPH` はメタデータ操作（要素テーブルへの read のみ・実測 `totalSlotMs=0`・
無課金）のため、冪等に再発行できます。

### 2-(A) が成功したか確認

```sql
SELECT property_graph_name
FROM `<MY_PROJECT>.<MY_DATASET>.INFORMATION_SCHEMA.PROPERTY_GRAPHS`
WHERE property_graph_name = 'politics'
```

1 行返れば成功。**ステップ 3 へ進む**。

権限・参照不可で失敗する場合（例: linked dataset への read-only 参照が PROPERTY GRAPH の要素テーブルとして
認められない等）は、**ステップ 2-(B) のフォールバックへ**。

---

## ステップ 2-(B): CTAS フォールバック（linked dataset 参照が不可だった場合）

> **背景**: 「linked dataset 上で購読者側 `CREATE PROPERTY GRAPH` が動くか」は購読者環境で保証しません。
> 権限や参照制約で 2-(A) が失敗した場合に備え、購読者が**リンク先テーブルを自 dataset へ CTAS で
> 物理コピー**してから PROPERTY GRAPH を定義する代替手順を用意しています。

### B-1. 32 テーブルを CTAS で物理コピー

linked dataset の全要素テーブル（ノード 9 + エッジ 23 = 32 本）を `<MY_DATASET>` にコピーする。
`bq` でループ実行する例:

```bash
PROJECT=<MY_PROJECT>
LINKED=<LINKED_DATASET>
DST=<MY_DATASET>
LOC=asia-northeast1

TABLES=(
  nodes_politician nodes_political_party nodes_parliamentary_group nodes_governing_body
  nodes_conference nodes_meeting nodes_proposal nodes_election nodes_vote_event
  edges_member_of edges_member_of_conference edges_affiliated_with edges_ran_in
  edges_spoke_in edges_voted_on edges_group_voted_on edges_submitted
  edges_group_submitted edges_conference_submitted edges_part_of edges_belongs_to_gb
  edges_held_by edges_held_for edges_composed_of edges_discussed_in edges_deliberated_by
  edges_decides edges_held_at edges_cast_vote edges_group_position
  edges_presided edges_attended
)

for t in "${TABLES[@]}"; do
  bq query --location=$LOC --use_legacy_sql=false \
    "CREATE OR REPLACE TABLE \`$PROJECT.$DST.$t\` AS
     SELECT * FROM \`$PROJECT.$LINKED.$t\`"
done
```

> **コスト注意**: `bq cp` はメタデータのみのコピーでスキャン課金が発生しない（安価）が、Analytics Hub の
> linked（read-only 共有）データセットを**コピー元にできない場合がある**。そのため本手順では確実に動く CTAS を
> 既定にしている。CTAS は要素テーブル実サイズ分のスキャン課金が発生する（一度きり）。安価に済ませたい場合は
> まず `bq cp` を試し、linked ソース不可のときだけ上記 CTAS にフォールバックするとよい。
> データ更新（Publisher の nightly 反映）を取り込むには、このコピーを定期的に再実行する。

### B-2. コピーしたテーブルを参照して PROPERTY GRAPH を定義

ステップ 2-(A) の DDL の `<LINKED_DATASET>` を**すべて `<MY_DATASET>`**（コピー先）に読み替えて発行する。
それ以外（`REFERENCES`・`KEY`・`PROPERTIES`）は完全に同一。コピー後はテーブルが購読者の通常テーブルに
なるため、PROPERTY GRAPH の要素テーブルとして確実に参照できる。

---

## ステップ 3: クエリ実行

PROPERTY GRAPH ができたら GQL を実行できる。GQL を使わない（使えない）場合は、同じ結果をプレーン SQL でも
取得できる（[README](README.md)・各クエリ doc は両形式を併記）。

### GQL クエリ例（会派 → 議案の採決を辿る）

```sql
GRAPH `<MY_PROJECT>.<MY_DATASET>.politics`
MATCH (g:ParliamentaryGroup)-[v:GROUP_VOTED_ON]->(p:Proposal)
RETURN g.name AS parliamentary_group, v.judgment, COUNT(*) AS proposals
GROUP BY parliamentary_group, judgment
ORDER BY proposals DESC
LIMIT 20
```

> GQL の `RETURN` は **`AS` 必須**（`RETURN COUNT(*) c` は構文エラー）。`ANY SHORTEST` はパス変数束縛
> （`MATCH p = ANY SHORTEST ...`）が必要。

### プレーン SQL クエリ例（同じ結果・PROPERTY GRAPH 不要）

PROPERTY GRAPH を作らない購読者でも、linked dataset（または CTAS コピー先）を直接 JOIN すれば同じ結果が
得られる。`<DATASET>` は `<LINKED_DATASET>` または `<MY_DATASET>` のいずれか:

```sql
SELECT g.name AS parliamentary_group, v.judgment, COUNT(*) AS proposals
FROM `<MY_PROJECT>.<DATASET>.edges_group_voted_on` AS v
JOIN `<MY_PROJECT>.<DATASET>.nodes_parliamentary_group` AS g
  ON v.source_sid = g.sagebase_id
JOIN `<MY_PROJECT>.<DATASET>.nodes_proposal` AS p
  ON v.dest_sid = p.sagebase_id
GROUP BY parliamentary_group, judgment
ORDER BY proposals DESC
LIMIT 20
```

代表クエリ 5 種（投票類似度 / 共同提出 / 造反分析 / 審議経路 / as-of スナップショット）の
GQL・プレーン SQL 両形式は [README](README.md) と `01`〜`05` の各 doc を参照。

---

## データカバレッジ（公開時点の注記）

個人レベルのエッジは段階拡充中です。**最新のカバレッジ状況は [README](README.md) の
「データカバレッジ注記」に一元化しています**。

> 運用上の注意: `edges_spoke_in` は speaker→politician 名寄せ由来でカバレッジが低めです。
> 利用時は各行の `min_matching_confidence` / `all_verified` を WHERE で必ず使ってください。

---

## 関連ドキュメント

- 代表クエリ集 README（GQL + プレーン SQL 併記）: [README](README.md)
- 購読手順: [../../README.md#購読手順bigquery-console](../../README.md#購読手順bigquery-console)
