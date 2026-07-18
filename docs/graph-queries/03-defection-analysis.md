# 03. 会派方針との乖離（造反）分析

会派の方針に反して投票した議員（造反）を、`VOTED_ON.is_defection` フラグと、その採決時点で
所属していた会派（`VOTED_ON.parliamentary_group_sid`）の**期間整合**で確定する。

- **想定購読者**: 政治記者、選挙アナリスト（党議拘束の実態分析）
- **主役エッジ**: `VOTED_ON`（個人・時点型）× `MEMBER_OF`（期間型）
- **設計原則**:
  - 造反の文脈となる会派は `VOTED_ON.parliamentary_group_sid`（採決時点の所属会派）を使う
  - `MEMBER_OF` の期間述語 `voted_date BETWEEN start_date AND COALESCE(end_date, '9999-12-31')` で
    「その採決日に本当にその会派に所属していたか」を検証する（時点型 × 期間型の整合）

---

## データ状況

参議院の記名投票（`source_type=ROLL_CALL`）と、会派採決の個人展開（`GROUP_EXPANSION`）を
`edges_voted_on` に投入済みです（現状 132,331 行・国会スコープ中心）。

**造反フラグ `is_defection` は現時点で全行 NULL** です（会派方針の確定処理の本番反映を進めています）。
下記の v1/v2 クエリは `is_defection = TRUE` を条件に使うため、現時点では**両方とも空を返します**。
本番反映後は同じクエリがそのまま結果を返します。

造反クエリは **2 系統** を併載します:

- **v1（membership 不要）**: `VOTED_ON.is_defection` を直接引く。会派 membership が不完全でも
  造反議員を一覧できる。
- **v2（正準・期間整合）**: `MEMBER_OF` の期間述語で「採決日に在籍した会派」を検証する完全形。

---

## v1 造反クエリ（membership 不要）

`is_defection = TRUE` の投票を、採決時点の会派（`parliamentary_group_sid`）とともに一覧します。
`MEMBER_OF` を介さないため membership 未投入でも造反議員が返る設計です。
`parliamentary_group_sid` は歴史的会派（現行会派名に無い名称）では NULL になりうるため
`LEFT JOIN` で結んでいます。

### プレーン SQL

```sql
SELECT pol.name AS politician, pg.name AS group_at_vote, prop.title,
       v.voted_date, v.approve
FROM `sagebase-gcp.sagebase_graph.edges_voted_on` AS v
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON v.source_sid = pol.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON v.dest_sid = prop.sagebase_id
LEFT JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg
       ON v.parliamentary_group_sid = pg.sagebase_id
WHERE v.is_defection = TRUE
ORDER BY v.voted_date DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[v:VOTED_ON]->(prop:Proposal)
WHERE v.is_defection = TRUE
RETURN pol.name AS politician, v.parliamentary_group_sid AS group_at_vote_sid,
       prop.title, v.voted_date, v.approve
ORDER BY v.voted_date DESC
LIMIT 20
```

**結果**: **現時点では 0 行**（`is_defection` 全 NULL のため）。会派方針の確定処理を本番 BQ に
反映した後、同じクエリで造反議員一覧が返ります。会派方針＝採決時点の会派内多数決に反した個人。
無所属・各派に属しない議員は党議拘束が無いため方針 None（造反対象外）として扱います。

---

## v2 造反クエリ（正準・期間整合・MEMBER_OF 在籍検証）

会派文脈を `MEMBER_OF` の期間述語で検証する完全形。`is_defection = TRUE` の投票を、採決日に
当該会派へ所属していた `MEMBER_OF` 行と突き合わせ、`parliamentary_group_sid` と `MEMBER_OF.dest_sid`
を一致させて「どの会派の方針に反したか」を在籍期間込みで確定する。

> ⚠️ **limitation**: 参議院の `parliamentary_group_memberships` は未投入のため、本クエリの
> 参議院分は現時点で **0 行**です。参議院 membership の投入と、`is_defection` の本番反映の
> 両方が完了した後に v2 も点灯します。衆議院・地方の membership がある会派では
> `is_defection` の反映後に点灯します。

### プレーン SQL

```sql
SELECT pol.name AS politician, pg.name AS group_at_vote, prop.title, v.voted_date, v.approve
FROM `sagebase-gcp.sagebase_graph.edges_voted_on` AS v
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON v.source_sid = pol.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON v.dest_sid = prop.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg ON v.parliamentary_group_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_member_of` AS m
  ON m.source_sid = v.source_sid AND m.dest_sid = v.parliamentary_group_sid
  AND v.voted_date BETWEEN m.start_date AND COALESCE(m.end_date, DATE '9999-12-31')
WHERE v.is_defection = TRUE
ORDER BY v.voted_date DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[v:VOTED_ON]->(prop:Proposal)
WHERE v.is_defection = TRUE
MATCH (pol)-[m:MEMBER_OF]->(pg:ParliamentaryGroup)
WHERE pg.sagebase_id = v.parliamentary_group_sid
  AND v.voted_date BETWEEN m.start_date AND COALESCE(m.end_date, DATE '9999-12-31')
RETURN pol.name AS politician, pg.name AS group_at_vote, prop.title, v.voted_date, v.approve
ORDER BY v.voted_date DESC
LIMIT 20
```

**結果**: **現時点では 0 行**（`is_defection` 全 NULL + 参議院 membership 未投入）。
両方の反映後に v1 と同じ造反集合へ収束します。

---

## 同型クエリ（会派採決 × 採決日在籍議員・実データ）

造反の前提となる**「採決日にその会派へ在籍していた議員集合」**を、`GROUP_VOTED_ON`（会派の判断）と
`MEMBER_OF`（期間型）の期間整合で実データ算出する。造反検出と**同じ期間述語**
（`voted_date BETWEEN start_date AND COALESCE(end_date, '9999-12-31')`）を使う。

> `MEMBER_OF` は 1 議員に複数行を持ちうる（任期更新等）ため、在籍議員数は必ず
> `COUNT(DISTINCT source_sid)` で数える。

### プレーン SQL

```sql
SELECT pg.name AS group_name, prop.title, gv.judgment, gv.voted_date,
  COUNT(DISTINCT m.source_sid) AS active_members_on_vote_date
FROM `sagebase-gcp.sagebase_graph.edges_group_voted_on` AS gv
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg ON gv.source_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON gv.dest_sid = prop.sagebase_id
LEFT JOIN `sagebase-gcp.sagebase_graph.edges_member_of` AS m
  ON m.dest_sid = gv.source_sid
  AND gv.voted_date BETWEEN m.start_date AND COALESCE(m.end_date, DATE '9999-12-31')
WHERE gv.voted_date = DATE '2024-12-12'
GROUP BY group_name, prop.title, gv.judgment, gv.voted_date
ORDER BY active_members_on_vote_date DESC, group_name
LIMIT 20
```

### 結果サンプル（先頭・国会 2024-12-12）

| group_name | title | judgment | voted_date | active_members_on_vote_date |
|------------|-------|----------|------------|-----------------------------|
| 公明党 | 防衛省の職員の給与等に関する法律の一部を改正する法律案 | 賛成 | 2024-12-12 | 53 |
| 公明党 | 令和六年度一般会計補正予算（第１号） | 賛成 | 2024-12-12 | 53 |
| 公明党 | 検察官の俸給等に関する法律の一部を改正する法律案 | 賛成 | 2024-12-12 | 53 |
| 公明党 | 国家公務員の育児休業等に関する法律の一部を改正する法律案 | 賛成 | 2024-12-12 | 53 |
| 公明党 | 地方交付税法及び特別会計に関する法律の一部を改正する法律案 | 賛成 | 2024-12-12 | 53 |

> 「会派は賛成、所属議員は 53 名」という土台ができると、個人投票（`VOTED_ON`）投入後は
> 「53 名のうち反対した者 = 造反」を `judgment` と `approve` の突き合わせで導ける。本クエリは
> 造反分析の**期間整合の半分**を実データで先取り実証している。

---

## 実測メトリクス（`--use_cache=false`・asia-northeast1・オンデマンド）

| クエリ | 現時点の結果 | 備考 |
|--------|-------------|------|
| v1 SQL（造反・membership 不要） | 0 行 | `is_defection` が全行 NULL のため。本番反映後に点灯 |
| v2 SQL（正準・MEMBER_OF 期間整合） | 0 行 | `is_defection` + 参議院 membership 双方の反映後に点灯 |
| 同型 SQL（会派×在籍） | 数十行 | `GROUP_VOTED_ON` × `MEMBER_OF`。期間整合の半分を実データ実証 |

> 同型 SQL は `LEFT JOIN ... BETWEEN` の期間突合が行展開を伴うため slot_ms が大きめに出ますが、
> 課金バイト（40 MB 前後）と実行時間は小さく、オンデマンドで問題なく完走します。
> slot 値は実行ごとに変動します（数 slot 秒のオーダー）。
