# 01. 投票類似度ネットワーク

同じ議案に同じ賛否を投じた回数から、議員（または会派）間の**実効的な派閥構造**を浮かび上がらせる。
`VOTED_ON` の 2 ホップ射影（Politician → Proposal ← Politician）でコミュニティ検出の入力グラフを作る。

- **想定購読者**: データジャーナリスト、政治学研究者（投票行動のクラスタリング）
- **主役エッジ**: `VOTED_ON`（個人・時点型）。会派版は `GROUP_VOTED_ON`
- **設計原則**: キーは `sagebase_id`。賛否の一致は `approve`（個人）/ `judgment`（会派）で判定

---

## ⚠️ データ状況

個人投票エッジ `edges_voted_on` は投入済み（132,331 行・国会スコープ中心）ですが、
本クエリで使う `approve` は会派採決の個人展開（source_type=`GROUP_EXPANSION`）と参議院記名投票
（`ROLL_CALL`）を混在して扱うため、実質的な同調ネットワークとしては会派レベル `GROUP_VOTED_ON`
（16,174 行）で見るのが素直です。個人 `VOTED_ON` はサンプル SQL/GQL の構文検証としても
そのまま実行できます。

---

## 正準クエリ（個人レベル `VOTED_ON`）

同一議案・同一賛否のペアごとに一致回数 `agreements` を数える。`a.sagebase_id < b.sagebase_id` で
無向ペアの重複を防ぐ。

### プレーン SQL

```sql
SELECT
  pa.name AS politician_a,
  pb.name AS politician_b,
  COUNT(*) AS agreements
FROM `sagebase-gcp.sagebase_graph.edges_voted_on` AS a
JOIN `sagebase-gcp.sagebase_graph.edges_voted_on` AS b
  ON a.dest_sid = b.dest_sid
  AND a.approve = b.approve
  AND a.source_sid < b.source_sid
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pa ON a.source_sid = pa.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pb ON b.source_sid = pb.sagebase_id
GROUP BY politician_a, politician_b
ORDER BY agreements DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (a:Politician)-[v1:VOTED_ON]->(p:Proposal)<-[v2:VOTED_ON]-(b:Politician)
WHERE a.sagebase_id < b.sagebase_id AND v1.approve = v2.approve
RETURN a.name AS politician_a, b.name AS politician_b, COUNT(*) AS agreements
ORDER BY agreements DESC
LIMIT 20
```

**結果**: `edges_voted_on` の投入分（国会スコープ）に対して構文上そのまま走ります。
地方議会レベルの拡充後は同じクエリで自治体横断のペア分析ができます。

---

## 同型クエリ（会派レベル `GROUP_VOTED_ON`・実データ）

`GROUP_VOTED_ON` で会派同士の同調採決数を数える。会派名は自治体をまたいで重複しうるため、
`BELONGS_TO_GB` で**国会**にスコープして解釈可能にしている（議案は単一自治体に属するため、
ペアは自然に同一自治体内で閉じる）。

### プレーン SQL

```sql
WITH kokkai_groups AS (
  SELECT bg.source_sid AS group_sid
  FROM `sagebase-gcp.sagebase_graph.edges_belongs_to_gb` AS bg
  JOIN `sagebase-gcp.sagebase_graph.nodes_governing_body` AS gb
    ON bg.dest_sid = gb.sagebase_id
  WHERE gb.name = '国会'
)
SELECT
  ga.name AS group_a,
  gb_.name AS group_b,
  COUNT(*) AS co_votes
FROM `sagebase-gcp.sagebase_graph.edges_group_voted_on` AS a
JOIN `sagebase-gcp.sagebase_graph.edges_group_voted_on` AS b
  ON a.dest_sid = b.dest_sid
  AND a.judgment = b.judgment
  AND a.source_sid < b.source_sid
JOIN kokkai_groups AS ka ON a.source_sid = ka.group_sid
JOIN kokkai_groups AS kb ON b.source_sid = kb.group_sid
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS ga ON a.source_sid = ga.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_parliamentary_group` AS gb_ ON b.source_sid = gb_.sagebase_id
GROUP BY group_a, group_b
ORDER BY co_votes DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (a:ParliamentaryGroup)-[:BELONGS_TO_GB]->(gb:GoverningBody {name: "国会"}),
      (a)-[v1:GROUP_VOTED_ON]->(p:Proposal)<-[v2:GROUP_VOTED_ON]-(b:ParliamentaryGroup),
      (b)-[:BELONGS_TO_GB]->(gb)
WHERE a.sagebase_id < b.sagebase_id AND v1.judgment = v2.judgment
RETURN a.name AS group_a, b.name AS group_b, COUNT(*) AS co_votes
ORDER BY co_votes DESC
LIMIT 20
```

### 結果サンプル（先頭 12 行・SQL/GQL 一致）

| group_a | group_b | co_votes |
|---------|---------|----------|
| 自由民主党・無所属の会 | 公明党 | 741 |
| 公明党 | 日本維新の会 | 721 |
| 公明党 | 国民民主党・無所属クラブ | 700 |
| 公明党 | 日本共産党 | 664 |
| 公明党 | 自由民主党 | 633 |
| 国民民主党・無所属クラブ | 日本維新の会 | 548 |
| 日本共産党 | 国民民主党・無所属クラブ | 527 |
| 国民民主党・無所属クラブ | 立憲民主党・無所属 | 519 |
| 社会民主党・市民連合 | 日本共産党 | 494 |
| 公明党 | 立憲民主党・無所属 | 478 |
| 自由民主党・無所属の会 | 国民民主党・無所属クラブ | 472 |
| 日本共産党 | 日本維新の会 | 465 |

> 与党ブロック（自民・公明）の同調が最上位に来る一方、野党間の同調も高い。これは多数の議案
> （人事・歳費・予算の形式的議決等）で全会派が「賛成」を投じるためで、対立軸を見るには `judgment` 別・
> 議案カテゴリ別の正規化（同調率 = 同調数 / 共通投票数）が次の一手になる。

この会派ネットワークの可視化は [visualization.md](visualization.md#viz1) を参照。

---

## 実測メトリクス（`--use_cache=false`・asia-northeast1・オンデマンド）

| クエリ | slot_ms（目安） | 課金バイト |
|--------|-----------------|-----------|
| 正準 SQL（VOTED_ON） | 数十〜数百 | 20〜30 MB |
| 正準 GQL（VOTED_ON） | 数十〜数百 | 250 MB（固定） |
| 同型 SQL（GROUP_VOTED_ON） | 数十〜数百 | 40 MB 前後 |
| 同型 GQL（GROUP_VOTED_ON） | 数十〜数百 | 250 MB（固定） |

> GQL は結果に関係なく課金バイトが 250 MB 固定（25 要素テーブル × 10 MB 下限）。SQL は触れた
> 4〜6 テーブルのみで 20〜40 MB。**同一目的なら SQL の方が安く速い**。
