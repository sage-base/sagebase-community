# 02. 共同提出ネットワーク

同じ議案を共同提出した議員ペアから、**立法上の協働関係**を射影する。`SUBMITTED` の 2 ホップ
射影（Politician → Proposal ← Politician）。投票類似度（[01](01-voting-similarity.md)）と同じ
「共有ノードを介した politician-politician 射影」パターンである。

- **想定購読者**: 政治学研究者（議員連携の定量化）、市民オンブズマン
- **主役エッジ**: `SUBMITTED`（個人・時点型）
- **設計原則**: キーは `sagebase_id`。`is_representative`（代表提出者）で筆頭提出者を区別できる

---

## ⚠️ データ状況

個人提出エッジ `edges_submitted` は**現状 0 行**です。提出者マスタ `proposal_submitters` は
投入されていますが、提出者 `raw_name` と政治家 FK の紐付け（名寄せ）を本番 BQ に反映中のため、
両端ノードを満たす行がまだありません。実データで射影パターンを示すため、同型の 2 ホップ
自己結合を**実データのある `MEMBER_OF`（共同所属ネットワーク）**で併載します。

---

## 正準クエリ（`SUBMITTED` 共同提出）

### プレーン SQL

```sql
SELECT pa.name AS politician_a, pb.name AS politician_b, COUNT(*) AS co_submissions
FROM `sagebase-gcp.sagebase_graph.edges_submitted` AS a
JOIN `sagebase-gcp.sagebase_graph.edges_submitted` AS b
  ON a.dest_sid = b.dest_sid AND a.source_sid < b.source_sid
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pa ON a.source_sid = pa.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pb ON b.source_sid = pb.sagebase_id
GROUP BY politician_a, politician_b
ORDER BY co_submissions DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (a:Politician)-[:SUBMITTED]->(p:Proposal)<-[:SUBMITTED]-(b:Politician)
WHERE a.sagebase_id < b.sagebase_id
RETURN a.name AS politician_a, b.name AS politician_b, COUNT(*) AS co_submissions
ORDER BY co_submissions DESC
LIMIT 20
```

**結果**: 現時点で 0 行（`edges_submitted` が空のため）。名寄せの本番反映後は同じクエリで結果を返します。

---

## 同型クエリ（`MEMBER_OF` 共同所属・実データ）

同一会派に**期間が重なって**在籍した議員ペアを射影する。`SUBMITTED` と同じ
politician-politician 2 ホップ射影に、**期間型エッジの重複判定**を加えた形。

期間の重複条件は「区間 A の開始 ≤ 区間 B の終了」かつ「区間 B の開始 ≤ 区間 A の終了」。
`end_date` の NULL（在任中）は **2 通りに補完し分ける**: 重複判定では番兵 `9999-12-31`（端点を開く）、
表示用の共同在籍日数 `co_tenure_days` の算出では doc 基準日 `2026-06-15` で打ち切る（在任中ペアの日数過大計上を避ける）。

### プレーン SQL

```sql
SELECT
  pa.name AS politician_a, pb.name AS politician_b, g.name AS shared_group,
  DATE_DIFF(
    LEAST(COALESCE(a.end_date, DATE '2026-06-15'), COALESCE(b.end_date, DATE '2026-06-15')),
    GREATEST(a.start_date, b.start_date), DAY) AS co_tenure_days
FROM `sagebase-gcp.sagebase_graph.edges_member_of` AS a
JOIN `sagebase-gcp.sagebase_graph.edges_member_of` AS b
  ON a.dest_sid = b.dest_sid AND a.source_sid < b.source_sid
  AND a.start_date <= COALESCE(b.end_date, DATE '9999-12-31')
  AND b.start_date <= COALESCE(a.end_date, DATE '9999-12-31')
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS g ON a.dest_sid = g.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pa ON a.source_sid = pa.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pb ON b.source_sid = pb.sagebase_id
ORDER BY co_tenure_days DESC, politician_a, politician_b
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (a:Politician)-[m1:MEMBER_OF]->(g:ParliamentaryGroup)<-[m2:MEMBER_OF]-(b:Politician)
WHERE a.sagebase_id < b.sagebase_id
  AND m1.start_date <= COALESCE(m2.end_date, DATE '9999-12-31')
  AND m2.start_date <= COALESCE(m1.end_date, DATE '9999-12-31')
RETURN a.name AS politician_a, b.name AS politician_b, g.name AS shared_group,
  DATE_DIFF(
    LEAST(COALESCE(m1.end_date, DATE '2026-06-15'), COALESCE(m2.end_date, DATE '2026-06-15')),
    GREATEST(m1.start_date, m2.start_date), DAY) AS co_tenure_days
ORDER BY co_tenure_days DESC, politician_a, politician_b
LIMIT 20
```

### 結果サンプル（先頭 12 行・SQL/GQL 一致）

| politician_a | politician_b | shared_group | co_tenure_days |
|--------------|--------------|--------------|----------------|
| 井上 義久 | 大口 善徳 | 公明党 | 6133 |
| 井上 義久 | 斉藤 鉄夫 | 公明党 | 6133 |
| 井上 義久 | 東 順治 | 公明党 | 6133 |
| 井上 義久 | 漆原 良夫 | 公明党 | 6133 |
| 井上 義久 | 石田 祝稔 | 公明党 | 6133 |
| 井上 義久 | 高木 美智代 | 公明党 | 6133 |
| 佐々木 憲昭 | 塩川鉄也 | 日本共産党 | 6133 |
| 佐々木 憲昭 | 志位和夫 | 日本共産党 | 6133 |
| 佐々木 憲昭 | 竹山 祐樹 | 日本共産党 | 6133 |
| 佐々木 憲昭 | 笠井 亮 | 日本共産党 | 6133 |
| 佐々木 憲昭 | 赤嶺 政賢 | 日本共産党 | 6133 |
| 坂口 力 | 井上 義久 | 公明党 | 6133 |

> 同一会派の長期在籍ペアが上位に並ぶ。`co_tenure_days` が同値で並ぶのは、`end_date` が NULL（在任中）の
> メンバー同士が doc 基準日まで一律に算出されるため。会派ごとに視覚的なクラスタを形成する
> （[visualization.md](visualization.md#viz2) の共同所属ネットワーク参照）。源泉名寄せ後は同じ自己結合構造を
> `edges_submitted` に置き換えるだけで真の共同提出ネットワークになります。

---

## 実測メトリクス（`--use_cache=false`・asia-northeast1・オンデマンド）

| クエリ | slot_ms（目安） | 課金バイト |
|--------|-----------------|-----------|
| 正準 SQL（SUBMITTED） | 数十〜数百 | 20 MB 前後 |
| 正準 GQL（SUBMITTED） | 数十〜数百 | 250 MB（固定） |
| 同型 SQL（MEMBER_OF） | 数十〜数百 | 30 MB 前後 |
| 同型 GQL（MEMBER_OF） | 数十〜数百 | 250 MB（固定） |
