# 07. VoteEvent 経由の審議経路 + 採決

議案の採決を Popolo `Motion → VoteEvent → Vote` 連鎖に倣って **VoteEvent (採決) ノード**として reify した
新モデルです。委員会採決と本会議採決を別 VoteEvent として分離し、
「同じ議案に複数採決がある」実データ（両採決を持つ議案が数千件規模）を辿れるようにしています。

- **想定購読者**: 立法過程研究者、政治学者（「委員会採決と本会議採決で誰が投票行動を変えたか」の分析）
- **主役ノード**: `VoteEvent`（`stage` = 本会議採決 / 委員会採決）
- **主役エッジ**:
  - `VoteEvent -[DECIDES]-> Proposal`
  - `VoteEvent -[HELD_AT]-> Meeting`（meeting 紐付けありの採決のみ）
  - `Politician -[CAST_VOTE]-> VoteEvent`（`attribution_rule` 付き）
  - `ParliamentaryGroup -[GROUP_POSITION]-> VoteEvent`（`attribution_rule` 付き）

## 既存エッジとの関係

既存の `VOTED_ON` / `GROUP_VOTED_ON` は購読者互換のため**外形不変で温存**しています（件数・スキーマとも変更なし）。
本モデルで追加した VoteEvent 系エッジは、その一段深い Popolo 表現を提供する併存レイヤーです。

- `Politician -[VOTED_ON]-> Proposal`: 全ての個人投票を含む従来型直エッジ（VoteEvent 未登録の議案 votes も乗る）
- `Politician -[CAST_VOTE]-> VoteEvent -[DECIDES]-> Proposal`: VoteEvent 経由の一段深い表現（`proposal_deliberations` の採決 stage 行がある議案のみ）

VoteEvent 未登録議案（`proposal_deliberations` に本会議採決 / 委員会採決 の stage 行が無い議案）の
個人票は `CAST_VOTE` に現れず、従来通り `VOTED_ON` にのみ現れます。

## 帰属ルール (attribution_rule)

源泉 `proposal_judges` / `proposal_vote_records` には stage 列がありません。議案が両採決（本会議 + 委員会）を
持つ場合、票がどの VoteEvent に属するかを決める必要があります。sagebase では
**本会議採決を default 帰属**としています。

- `attribution_rule = 'sole_stage'`: 議案の VoteEvent が 1 個 → その唯一に紐付け
- `attribution_rule = 'plenary_default'`: 議案が両採決を持つ → 本会議採決へ帰属

委員会採決 VoteEvent には CAST_VOTE / GROUP_POSITION が張られません（票→本会議 帰属）。
委員会での投票内訳が将来源泉に stage 明示付きで来た時は、そちらを優先する余地を残しています。

---

## クエリ 1: 議案の採決を stage 別に列挙（両採決の分離を可視化）

同じ議案に複数採決がある実データを 1 行 1 VoteEvent で並べます。

### プレーン SQL

```sql
SELECT
    prop.title,
    ve.stage,
    ve.voted_date,
    ve.legislative_session,
    m.name AS meeting_name
FROM `sagebase-gcp.sagebase_graph.nodes_vote_event` AS ve
JOIN `sagebase-gcp.sagebase_graph.edges_decides` AS d ON d.source_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON d.dest_sid = prop.sagebase_id
LEFT JOIN `sagebase-gcp.sagebase_graph.edges_held_at` AS ha ON ha.source_sid = ve.sagebase_id
LEFT JOIN `sagebase-gcp.sagebase_graph.nodes_meeting` AS m ON ha.dest_sid = m.sagebase_id
WHERE prop.sagebase_id IN (
    -- 両採決（本会議採決 AND 委員会採決）を持つ議案に絞る
    SELECT d.dest_sid
    FROM `sagebase-gcp.sagebase_graph.edges_decides` AS d
    JOIN `sagebase-gcp.sagebase_graph.nodes_vote_event` AS ve2 ON d.source_sid = ve2.sagebase_id
    GROUP BY d.dest_sid
    HAVING COUNT(DISTINCT ve2.stage) >= 2
)
ORDER BY prop.title, ve.stage
LIMIT 20
```

### GQL

> 「両採決を持つ議案」を GQL で表現する場合、パターンで「同じ議案に本会議 VoteEvent と委員会
> VoteEvent が両方 DECIDES している」条件を書くのが素直です。パターンマッチ後に議案に紐づく VoteEvent
> を再度辿って明細行を返します。

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (plenary_ve:VoteEvent)-[:DECIDES]->(prop:Proposal)<-[:DECIDES]-(committee_ve:VoteEvent),
      (ve:VoteEvent)-[:DECIDES]->(prop)
OPTIONAL MATCH (ve)-[:HELD_AT]->(m:Meeting)
WHERE plenary_ve.stage = '本会議採決' AND committee_ve.stage = '委員会採決'
RETURN DISTINCT prop.title AS title, ve.stage AS stage, ve.voted_date AS voted_date,
       ve.legislative_session AS legislative_session, m.name AS meeting_name
ORDER BY title, stage
LIMIT 20
```

**期待**: 両採決を持つ議案の各行が「委員会採決 / 本会議採決」の 2 行に分離されます。

---

## クエリ 2: VoteEvent 経由で採決参加者を追跡（`attribution_rule` 付き）

議員が採決でどう投票したかを VoteEvent 経由で辿ります。`attribution_rule` を SELECT に含めることで、
帰属ルールの根拠を透明化します。

### プレーン SQL

```sql
SELECT
    prop.title,
    ve.stage,
    ve.voted_date,
    pol.name AS politician,
    cv.approve,
    cv.is_defection,
    cv.attribution_rule
FROM `sagebase-gcp.sagebase_graph.edges_cast_vote` AS cv
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON cv.source_sid = pol.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_vote_event` AS ve ON cv.dest_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_decides` AS d ON d.source_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON d.dest_sid = prop.sagebase_id
WHERE ve.stage = '本会議採決'
ORDER BY ve.voted_date DESC, prop.title, politician
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[cv:CAST_VOTE]->(ve:VoteEvent)-[:DECIDES]->(prop:Proposal)
WHERE ve.stage = '本会議採決'
RETURN
    prop.title,
    ve.stage,
    ve.voted_date,
    pol.name AS politician,
    cv.approve,
    cv.is_defection,
    cv.attribution_rule
ORDER BY ve.voted_date DESC, prop.title, politician
LIMIT 20
```

**期待**: `attribution_rule = 'sole_stage'`（議案が本会議採決のみを持つ）と `'plenary_default'`（両採決を持ち、
本会議へ default 帰属）が混在します。

---

## クエリ 3: 会派方針と個人票の照合（GROUP_POSITION × CAST_VOTE）

同じ VoteEvent に対する会派方針と個人票を突合します（造反分析の Popolo 型実装）。

### プレーン SQL

```sql
SELECT
    prop.title,
    ve.stage,
    ve.voted_date,
    pg.name AS parliamentary_group,
    gp.judgment AS group_position,
    pol.name AS politician,
    cv.approve AS politician_vote,
    cv.is_defection
FROM `sagebase-gcp.sagebase_graph.edges_group_position` AS gp
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS pg ON gp.source_sid = pg.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_vote_event` AS ve ON gp.dest_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_decides` AS d ON d.source_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON d.dest_sid = prop.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_cast_vote` AS cv ON cv.dest_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON cv.source_sid = pol.sagebase_id
WHERE cv.parliamentary_group_sid = pg.sagebase_id  -- 同じ会派の議員のみ
ORDER BY ve.voted_date DESC, prop.title, pg.name, politician
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pg:ParliamentaryGroup)-[gp:GROUP_POSITION]->(ve:VoteEvent)-[:DECIDES]->(prop:Proposal),
      (pol:Politician)-[cv:CAST_VOTE]->(ve)
WHERE cv.parliamentary_group_sid = pg.sagebase_id
RETURN
    prop.title,
    ve.stage,
    ve.voted_date,
    pg.name AS parliamentary_group,
    gp.judgment AS group_position,
    pol.name AS politician,
    cv.approve AS politician_vote,
    cv.is_defection
ORDER BY ve.voted_date DESC, prop.title, pg.name, politician
LIMIT 20
```

**期待**: 各行は「同一 VoteEvent × 同一会派 × その会派所属の議員個票」の突合です。`is_defection = TRUE` は
会派方針と異なる投票をした議員行（造反）。

---

## 検証: VoteEvent 分離の確認クエリ

「同一議案に複数採決がある実データで VoteEvent が分離されること」を確認するクエリです。

```sql
SELECT
    prop.title,
    ARRAY_AGG(ve.stage ORDER BY ve.stage) AS stages,
    ARRAY_AGG(ve.sagebase_id ORDER BY ve.stage) AS vote_event_sids,
    COUNT(*) AS vote_event_count
FROM `sagebase-gcp.sagebase_graph.nodes_vote_event` AS ve
JOIN `sagebase-gcp.sagebase_graph.edges_decides` AS d ON d.source_sid = ve.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON d.dest_sid = prop.sagebase_id
GROUP BY prop.sagebase_id, prop.title
HAVING COUNT(*) >= 2
ORDER BY vote_event_count DESC, prop.title
LIMIT 20
```

**期待**: `stages = ['委員会採決', '本会議採決']` の議案（両採決を持つ）が並び、それぞれ別の `vote_event_sid`
になっている（VoteEvent 分離の証跡）。

---

## 関連

- 既存 VOTED_ON / GROUP_VOTED_ON クエリ: [01](01-voting-similarity.md) / [03](03-defection-analysis.md)
- 審議経路 (DISCUSSED_IN / DELIBERATED_BY): [04](04-deliberation-path.md)
