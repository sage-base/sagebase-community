# 08. 議長経験と出席率（PRESIDED / ATTENDED）

議事録冒頭（国会形式）から抽出した会議単位の議長/委員長/理事関係と、
出席/欠席関係を辿ります。`Politician -[PRESIDED]-> Meeting` と
`Politician -[ATTENDED]-> Meeting` の 2 つのエッジ。

- **想定購読者**: 政治学研究者、報道機関（キャリア分析・活動量比較）
- **主役エッジ**: `PRESIDED`（時点型、`position` 属性付き）/ `ATTENDED`（時点型、
  `status` 属性付き）
- **設計原則**: 議事録冒頭の一次抽出源 → 名寄せ → 公開の 4 層アーキテクチャ。
  名寄せ品質は `matching_confidence` を必ず属性として付与し、低信頼マッチも
  属性で透明化します（閾値除外は網羅性と透明性を損なうため = SPOKE_IN と同じポリシー）。

---

## ⚠️ データ状況（Phase 1 = 国会のみ）

`edges_presided` / `edges_attended` は **Phase 1 として国会（kokkai）のみ公開**しています。
- 地方議会は minutes 本文の抽出源（gcs_text_uri）が未整備のため今後対応します。
- **`status='absent'`（明示的欠席）**は他データソースでは殆ど取れない差別化データですが、
  国会議事録の conversations には「欠席委員」セクションの本文がほぼ含まれない
  （冒頭本文が会議録情報にのみ集約されているため）ため、現状は `status='present'` のみで
  運用しています。将来 raw HTML 経路で復活する余地があります。

---

## 正準クエリ 1: 議長経験のある議員トップ 10

### プレーン SQL

```sql
-- 委員長経験の多い議員トップ 10 (Phase 1 は国会のみ)
SELECT
    pol.name,
    pol.furigana,
    COUNT(DISTINCT pr.dest_sid) AS meetings_chaired,
    MIN(pr.meeting_date) AS first_chaired,
    MAX(pr.meeting_date) AS last_chaired
FROM `sagebase-gcp.sagebase_graph.edges_presided` AS pr
JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol
    ON pr.source_sid = pol.sagebase_id
WHERE pr.position = 'chair'
  AND pr.matching_confidence >= 0.7   -- prefix_fuzzy 以上に絞る
GROUP BY pol.name, pol.furigana
ORDER BY meetings_chaired DESC
LIMIT 10
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[pr:PRESIDED {position: 'chair'}]->(m:Meeting)
WHERE pr.matching_confidence >= 0.7
RETURN pol.name AS name, pol.furigana AS furigana,
       COUNT(DISTINCT m) AS meetings_chaired,
       MIN(pr.meeting_date) AS first_chaired,
       MAX(pr.meeting_date) AS last_chaired
ORDER BY meetings_chaired DESC
LIMIT 10
```

---

## 正準クエリ 2: 会議体別の出席率

### プレーン SQL

```sql
-- 委員会別 (会議体) の政治家別出席率 (Phase 1 は国会のみ)。
-- 出席 = ATTENDED エッジで status='present' が付いた回数。
-- 分母 = 政治家が MEMBER_OF_CONFERENCE で登録されている期間の会議数。
WITH per_conference AS (
    SELECT
        pol.name AS politician,
        conf.name AS conference,
        conf.sagebase_id AS conference_sid,
        pol.sagebase_id AS politician_sid,
        COUNT(DISTINCT m.sagebase_id) AS attended_count
    FROM `sagebase-gcp.sagebase_graph.edges_attended` AS a
    JOIN `sagebase-gcp.sagebase_graph.nodes_politician` AS pol
        ON a.source_sid = pol.sagebase_id
    JOIN `sagebase-gcp.sagebase_graph.nodes_meeting` AS m
        ON a.dest_sid = m.sagebase_id
    JOIN `sagebase-gcp.sagebase_graph.edges_held_by` AS hb
        ON hb.source_sid = m.sagebase_id
    JOIN `sagebase-gcp.sagebase_graph.nodes_conference` AS conf
        ON hb.dest_sid = conf.sagebase_id
    WHERE a.status = 'present'
      AND a.matching_confidence >= 0.7
    GROUP BY politician, conference, conference_sid, politician_sid
)
SELECT * FROM per_conference
ORDER BY attended_count DESC
LIMIT 20
```

### GQL

```sql
GRAPH `sagebase-gcp.sagebase_graph.politics`
MATCH (pol:Politician)-[a:ATTENDED {status: 'present'}]->(m:Meeting)-[:HELD_BY]->(conf:Conference)
WHERE a.matching_confidence >= 0.7
RETURN pol.name AS politician, conf.name AS conference,
       COUNT(DISTINCT m) AS attended_count
ORDER BY attended_count DESC
LIMIT 20
```

---

## 名寄せ品質フィルタの目安

| matching_confidence | 出所 | 使い方 |
|---|---|---|
| 1.00 | politicians.name または kanji_name の完全一致（候補 1 人） | 事実として使える |
| 0.70 | 姓 2 文字一致 + 先頭 3 文字以上共通の候補 1 人（prefix-fuzzy） | 集計向け（数字を出すには十分） |
| 0.50 | 完全一致だが同姓同名候補が複数（最小 id を暫定選択） | 個人特定には向かない |
| 0.00 | politician_id 未解決（no_match） | エッジには載らない（INNER JOIN で除外） |

**推奨**: 個人の主張として引用する場合は `matching_confidence >= 1.0` に絞ってください。
集計（件数・ランキング）には `>= 0.7` で十分です。`matching_confidence` を全く
参照せず使うと、名寄せノイズを事実として扱ってしまう可能性があります。

## 関連

- 抽出源: 議事録冒頭 8000 字の regex（国会形式）
- 名寄せ: politicians に対する 4 種分類（exact / prefix_fuzzy / ambiguous / no_match）
