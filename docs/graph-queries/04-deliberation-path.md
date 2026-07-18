# 04. 議案の審議経路追跡

議案が委員会から本会議へどう審議されたか、`stage`（付託・採決等の審議段階）の遷移を辿る。
`Proposal -[DISCUSSED_IN]-> Meeting -[HELD_BY]-> Conference` と
`Proposal -[DELIBERATED_BY]-> Conference` のチェーン走査。

- **想定購読者**: 立法過程研究者、政策ウォッチャー（議案のライフサイクル分析）
- **主役エッジ**: `DISCUSSED_IN` / `DELIBERATED_BY`（時点型・`stage` 属性付き）
- **設計原則**: `stage` は NULL を取りうるため、エッジキーは決定的ハッシュ `sagebase_id`
  （NULL を sentinel に畳む）で物理化済み（NULL キーによる走査脱落の回避）

---

## ⚠️ データ状況

`edges_discussed_in` / `edges_deliberated_by` は段階拡充中です（現状はそれぞれ 2 万件・11 万件程度）。
源泉 `proposal_deliberations` の投入が進み、全自治体カバレッジは今後さらに拡充します。
正準クエリはそのまま実行できますが、結果件数は投入状況に応じて増減します。
**多段の構造走査**の例として、審議経路が乗る**会場の階層構造**
（`Meeting -[HELD_BY]-> Conference -[PART_OF]-> GoverningBody`）を同型クエリとして併載します。

---

## 正準クエリ（審議経路）

### プレーン SQL

> GQL の `-[:HELD_BY]->(c:Conference)` は会議に `HELD_BY` エッジが在ることを要求する（内部結合）。
> SQL も `INNER JOIN` で揃えて両形式の結果を一致させている（`HELD_BY` を持たない会議は両形式とも落ちる）。

```sql
-- 議案の審議経路: Proposal -[DISCUSSED_IN]-> Meeting -[HELD_BY]-> Conference（stage 付き）
SELECT prop.title, di.stage, m.name AS meeting, m.date AS meeting_date, c.name AS conference
FROM `sagebase-gcp.sagebase_graph.edges_discussed_in` AS di
JOIN `sagebase-gcp.sagebase_graph.nodes_proposal` AS prop ON di.source_sid = prop.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_meeting` AS m ON di.dest_sid = m.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_held_by` AS hb ON hb.source_sid = m.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_conference` AS c ON hb.dest_sid = c.sagebase_id
ORDER BY prop.title, m.date
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (prop:Proposal)-[di:DISCUSSED_IN]->(m:Meeting)-[:HELD_BY]->(c:Conference)
RETURN prop.title, di.stage, m.name AS meeting, m.date AS meeting_date, c.name AS conference
ORDER BY prop.title, meeting_date
LIMIT 20
```

**結果**: 投入分に対して構文上そのまま走ります。全自治体カバレッジは順次拡充中のため、
自治体によっては該当議案の結果が空になることがあります。

---

## 同型クエリ（会場の階層構造・実データ）

審議経路が走査する**会議 → 会議体 → 自治体**の 3 階層を実データで辿る。`DISCUSSED_IN` 投入後は
この backbone の上に議案 → 会議のエッジが乗る。

### プレーン SQL

```sql
SELECT c.name AS conference, gb.name AS governing_body, COUNT(DISTINCT hb.source_sid) AS meetings
FROM `sagebase-gcp.sagebase_graph.edges_held_by` AS hb
JOIN `sagebase-gcp.sagebase_graph.nodes_conference` AS c ON hb.dest_sid = c.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.edges_part_of` AS po ON po.source_sid = c.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_governing_body` AS gb ON po.dest_sid = gb.sagebase_id
GROUP BY conference, governing_body
ORDER BY meetings DESC, conference
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (m:Meeting)-[:HELD_BY]->(c:Conference)-[:PART_OF]->(gb:GoverningBody)
RETURN c.name AS conference, gb.name AS governing_body, COUNT(DISTINCT m.sagebase_id) AS meetings
ORDER BY meetings DESC, conference
LIMIT 20
```

### 結果サンプル（先頭 12 行・SQL/GQL 一致）

| conference | governing_body | meetings |
|------------|----------------|----------|
| 衆議院議院運営委員会 | 国会 | 4755 |
| 衆議院本会議 | 国会 | 4707 |
| 参議院議院運営委員会 | 国会 | 3833 |
| 参議院本会議 | 国会 | 3674 |
| 衆議院法務委員会 | 国会 | 2702 |
| 予算特別委員会 | 北海道 | 2564 |
| 衆議院内閣委員会 | 国会 | 2553 |
| 衆議院農林水産委員会 | 国会 | 2416 |
| 衆議院予算委員会 | 国会 | 2413 |
| 衆議院大蔵委員会 | 国会 | 2359 |
| 参議院内閣委員会 | 国会 | 2298 |
| 定例会 | 北海道 | 2286 |

---

## 名寄せ品質フィルタの見本（`SPOKE_IN`・実データ）

`SPOKE_IN`（政治家 → 会議の発言集約）は**名寄せ由来エッジ**である。低信頼マッチを「事実」として
扱わないよう、`min_matching_confidence` / `all_verified` を **WHERE で必ず使う**のが設計上の約束。
ここでは審議の場（会議）での発言活動を、品質フィルタ付きで取得する。

> `SPOKE_IN` は名寄せ由来でカバレッジは低め（現状 3 万件超）。
> カバレッジは [README](README.md) の「データカバレッジ注記」に一元化しています。

### プレーン SQL

```sql
SELECT pol.name AS politician, m.name AS meeting, m.date AS meeting_date,
       s.speech_count, s.total_chars, s.min_matching_confidence, s.all_verified
FROM `sagebase-gcp.sagebase_graph.edges_spoke_in` AS s
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol ON s.source_sid = pol.sagebase_id
JOIN `sagebase-gcp.sagebase_graph.nodes_meeting` AS m ON s.dest_sid = m.sagebase_id
WHERE s.all_verified = TRUE OR s.min_matching_confidence >= 0.9
ORDER BY s.speech_count DESC, politician
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[s:SPOKE_IN]->(m:Meeting)
WHERE s.all_verified = TRUE OR s.min_matching_confidence >= 0.9
RETURN pol.name AS politician, m.name AS meeting, m.date AS meeting_date,
       s.speech_count, s.total_chars, s.min_matching_confidence, s.all_verified
ORDER BY s.speech_count DESC, politician
LIMIT 20
```

### 結果サンプル（先頭・SQL/GQL 一致）

| politician | meeting | meeting_date | speech_count | min_matching_confidence | all_verified |
|------------|---------|--------------|--------------|-------------------------|--------------|
| 川名康介 | 令和7年2月17日健康福祉常任委員会 | 2025-02-17 | 489 | 1 | false |
| 川名康介 | 令和7年6月19日健康福祉常任委員会 | 2025-06-19 | 467 | 1 | false |
| 松崎太洋 | 令和7年12月10日健康福祉常任委員会 | 2025-12-10 | 438 | 1 | false |
| 高橋祐子 | 令和7年2月19日県土整備常任委員会 | 2025-02-19 | 429 | 1 | false |
| 松崎太洋 | 令和7年9月25日健康福祉常任委員会 | 2025-09-25 | 408 | 1 | false |

> `min_matching_confidence = 1`（マッチ信頼度最高）でも `all_verified = false`（人手検証未済）の行が
> 並ぶ。両属性を公開することで、購読者は「自動名寄せのみ採用」「人手検証済みのみ採用」を自分で選べる。
> 恣意的な閾値除外をせず透明化する、という名寄せ品質の設計原則の現れ。

---

## 実測メトリクス（`--use_cache=false`・asia-northeast1・オンデマンド）

| クエリ | slot_ms（目安） | 課金バイト |
|--------|-----------------|-----------|
| 正準 SQL（審議経路） | 数十〜数百 | 50 MB 前後 |
| 正準 GQL（審議経路） | 数十〜数百 | 250 MB（固定） |
| 同型 SQL（会場階層） | 数百 | 40 MB 前後 |
| 同型 GQL（会場階層） | 数百 | 250 MB（固定） |
| SPOKE_IN SQL（品質フィルタ） | 数百 | 30 MB 前後 |
| SPOKE_IN GQL（品質フィルタ） | 数百 | 250 MB（固定） |
