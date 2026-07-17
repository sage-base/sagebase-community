# 05. as-of スナップショット

「**ある日付（YYYY-MM-DD）時点の会派構成と、その当日の採決**」を再構成する。期間型エッジ
（`MEMBER_OF`）の時間述語と、時点型エッジ（`VOTED_ON` / `GROUP_VOTED_ON`）の等値述語を組み合わせる、
**時間軸クエリの第一級の見本**。

- **想定購読者**: 時系列分析者、歴史研究者（過去時点の議会構成の復元）
- **主役エッジ**: `MEMBER_OF`（期間型）+ `VOTED_ON` / `GROUP_VOTED_ON`（時点型）
- **設計原則（時間軸の二分）**:
  - 期間型は `as_of_date BETWEEN start_date AND COALESCE(end_date, '9999-12-31')`
  - 時点型は `voted_date = as_of_date`
  - `MEMBER_OF` は 1 議員に複数行を持ちうるため、議員数は `COUNT(DISTINCT source_sid)`

アンカー: **国会 / 2024-12-12**（13 議案・130 会派採決のあった日）。

---

## Part A: as-of 会派構成（`MEMBER_OF`・実データ）

`2024-12-12` 時点で国会の各会派に在籍していた議員数を、期間述語で復元する。**このパートは実データで
完全に動作する**（`MEMBER_OF` は 2,455 行）。

### プレーン SQL

```sql
-- 国会 2024-12-12 時点の会派構成（期間述語で as-of スナップショット）
SELECT pg.name AS group_name, COUNT(DISTINCT m.source_sid) AS members
FROM `sagebase-gcp.sagebase_graph.edges_member_of` AS m
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg ON m.dest_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_belongs_to_gb` AS bg ON bg.source_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_governing_body` AS gb ON bg.dest_sid = gb.sagebase_id
WHERE gb.name = '国会'
  AND DATE '2024-12-12' BETWEEN m.start_date AND COALESCE(m.end_date, DATE '9999-12-31')
GROUP BY group_name
ORDER BY members DESC, group_name
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (p:Politician)-[m:MEMBER_OF]->(pg:ParliamentaryGroup)-[:BELONGS_TO_GB]->(gb:GoverningBody {name: "国会"})
WHERE DATE '2024-12-12' BETWEEN m.start_date AND COALESCE(m.end_date, DATE '9999-12-31')
RETURN pg.name AS group_name, COUNT(DISTINCT p.sagebase_id) AS members
ORDER BY members DESC, group_name
LIMIT 20
```

### 結果（国会 2024-12-12・SQL/GQL 一致）

| group_name | members |
|------------|---------|
| 公明党 | 53 |
| 自由民主党・無所属の会 | 35 |
| 立憲民主党・無所属 | 21 |
| 日本共産党 | 15 |
| 日本維新の会・教育無償化を実現する会 | 15 |
| れいわ新選組 | 3 |
| 国民民主党・無所属クラブ | 3 |
| 参政党 | 1 |
| 社会民主党・市民連合 | 1 |

> `COUNT(*)` だと自民が 81・公明が 80 と出るが、これは MEMBER_OF が 1 議員に複数行を持つため。
> `COUNT(DISTINCT source_sid)` で正しい人数（自民 35・公明 53）になる。as-of の人数集計では DISTINCT が必須。

---

## Part B: 当日の採決

> Part A と会派名で結合するため、Part B も Part A と**同じ国会スコープ**（`BELONGS_TO_GB` 経由）を
> 適用する。これを省くと、同一採決日に他の自治体の `GROUP_VOTED_ON` 行があった場合に会派名が衝突して
> 断面が混ざる（2024-12-12 は実際には国会のみだが、別日付・別アンカーでも安全なクエリにしておく）。

### 正準（個人投票 `VOTED_ON`・現状 0 行）

```sql
WITH kokkai_groups AS (
  SELECT bg.source_sid AS group_sid
  FROM `sagebase-gcp.sagebase_graph.edges_belongs_to_gb` AS bg
  JOIN `sagebase-gcp.sagebase_graph.nodes_governing_body` AS gb ON bg.dest_sid = gb.sagebase_id
  WHERE gb.name = '国会'
)
SELECT pol.name AS politician, prop.title, v.approve
FROM `sagebase-gcp.sagebase_graph.edges_voted_on` AS v
JOIN kokkai_groups AS k ON v.parliamentary_group_sid = k.group_sid
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON v.source_sid = pol.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON v.dest_sid = prop.sagebase_id
WHERE v.voted_date = DATE '2024-12-12'
ORDER BY prop.title, politician
LIMIT 20
```

**結果**: 個人投票の投入分に応じた結果を返します。国会スコープの投入が中心で、
地方議会レベルの拡充後は同じクエリで自治体横断の結果が取れます。

### 同型（会派採決 `GROUP_VOTED_ON`・実データ）

`GROUP_VOTED_ON` を使って**当日の会派採決**を返します。`voted_date = as_of_date` の時点述語に
国会スコープを加えます。

```sql
WITH kokkai_groups AS (
  SELECT bg.source_sid AS group_sid
  FROM `sagebase-gcp.sagebase_graph.edges_belongs_to_gb` AS bg
  JOIN `sagebase-gcp.sagebase_graph.nodes_governing_body` AS gb ON bg.dest_sid = gb.sagebase_id
  WHERE gb.name = '国会'
)
SELECT pg.name AS group_name, prop.title, gv.judgment, gv.member_count
FROM `sagebase-gcp.sagebase_graph.edges_group_voted_on` AS gv
JOIN kokkai_groups AS k ON gv.source_sid = k.group_sid
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg ON gv.source_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON gv.dest_sid = prop.sagebase_id
WHERE gv.voted_date = DATE '2024-12-12'
ORDER BY prop.title, group_name
LIMIT 20
```

```sql
-- GQL
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pg:ParliamentaryGroup)-[:BELONGS_TO_GB]->(gb:GoverningBody {name: "国会"}),
      (pg)-[gv:GROUP_VOTED_ON]->(prop:Proposal)
WHERE gv.voted_date = DATE '2024-12-12'
RETURN pg.name AS group_name, prop.title, gv.judgment, gv.member_count
ORDER BY prop.title, group_name
LIMIT 20
```

### 結果サンプル（先頭・SQL/GQL 一致）

| group_name | title | judgment | member_count |
|------------|-------|----------|--------------|
| れいわ新選組 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 反対 | NULL |
| 公明党 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 参政党 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 国民民主党・無所属クラブ | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 日本共産党 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 日本維新の会 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 立憲民主党・無所属 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |
| 自由民主党・無所属の会 | 一般職の職員の給与に関する法律等の一部を改正する法律案 | 賛成 | NULL |

> Part A（会派構成）と Part B（当日採決）を会派名で結べば「会派 X は当日 Y 件に賛成、その時点の
> 在籍は Z 名」という as-of の完全な断面になります。`member_count` が NULL なのは源泉
> `proposal_vote_records.member_count` が国会分で未記録のため（Part A の人数で代替可能）。

---

## 実測メトリクス（`--use_cache=false`・asia-northeast1・オンデマンド）

| クエリ | slot_ms（目安） | 課金バイト |
|--------|-----------------|-----------|
| Part A SQL（会派構成 as-of） | 数十 | 40 MB 前後 |
| Part A GQL（会派構成 as-of） | 数十 | 250 MB（固定） |
| Part B 正準 SQL（個人採決・国会 scoped） | 数十〜数百 | 50 MB 前後 |
| Part B 同型 SQL（会派採決・国会 scoped） | 数百 | 50 MB 前後 |
| Part B 同型 GQL（会派採決・国会 scoped） | 数百 | 250 MB（固定） |
