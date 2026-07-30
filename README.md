# sagebase community

政治活動データの修正・追加リクエストを受け付ける公開リポジトリです。

## sagebase とは

[sagebase](https://sage-base.com) は、日本の政治活動データ（国会・地方議会の議事録・議案・賛否・選挙結果など）を収集・構造化し、誰でもアクセスできる形で公開するプロジェクトです。

### 収集しているデータ

| データ種別 | 対象 | ソース |
|-----------|------|--------|
| 議事録・発言 | 衆議院・参議院 | 国会会議録検索システム |
| 議案・法案 | 衆議院・参議院 | 衆議院/参議院公式サイト |
| 議案賛否（本会議） | 衆議院・参議院 | 衆議院/参議院公式サイト |
| 選挙結果 | 衆院選・参院選 | 総務省選挙関連資料 |
| 政治家情報 | 国会議員 | 各政党公式サイト等 |
| 会派（院内会派） | 衆議院・参議院 | 国会公式サイト |
| 委員会構成 | 衆議院・参議院 | 国会公式サイト |
| 議事録・発言 | 47 都道府県議会 | 各議会サイト（kaigiroku.net / dbsr.jp / gijiroku ASP 等） |
| 議事録・発言 | 政令指定都市の一部 | 各議会サイト（順次拡充中） |
| 議案・議決 | 一部の都道府県議会 | 各議会サイト（対応議会は順次拡充） |

対象議会・データ範囲の詳細は [docs/data-coverage.md](docs/data-coverage.md) を参照してください。

## 公開データセット

BigQuery Analytics Hub 経由で **allAuthenticatedUsers 向けに公開**しています。Google アカウントを持つ全ユーザーが購読できます（クエリ課金は購読者側）。

| データセット | 種類 | 内容 |
|-------------|:---:|------|
| `sagebase-gcp.sagebase` | main 層 | 23 テーブル・安定版。政治家・議案・投票・議事録・発言など。 |
| `sagebase-gcp.sagebase_graph` | graph 層（Preview） | ノード 8 種・エッジ 17 種の graph-ready 形式。同じデータをネットワーク分析向けに再構築。 |

**Analytics Hub 経由でしか読めません**（データセット直参照はできません）。購読手順は下記のとおり:

### 購読手順（BigQuery Console）

1. [BigQuery Analytics Hub](https://console.cloud.google.com/bigquery/analytics-hub) を開く。
2. 上部の **「リスティングを検索」** で `sagebase` を検索。
3. 目的のリスティングを開く（安定版なら `sagebase_data`、graph 層なら `sagebase_graph_data`）。
4. 右上の **「サブスクライブ」** ボタンで、自分のプロジェクトに read-only の linked dataset を作成する。
5. 作成された linked dataset を参照してクエリを書ける（下記サンプル参照）。

> ロケーションは **`asia-northeast1`** 固定。linked dataset は購読時に自分で作成先プロジェクト・データセット名を指定できる。

コマンドラインの場合:

```bash
# Exchange とリスティングの一覧
bq --project_id=<自分のプロジェクト> --location=asia-northeast1 \
  ls --analytics_hub_exchange \
  --project_and_dataset_ids=sagebase-gcp:sagebase_exchange
```

Console GUI から購読するのが最も簡単です。

## 動作確認済みクエリ例

購読して linked dataset を `my_proj.sagebase_linked` に作った場合の例。自分のプロジェクト・データセット名に読み替えてください。

### 直近 5 件の会議を新しい順に

```sql
SELECT
  m.name AS meeting,
  m.date AS meeting_date,
  gb.name AS governing_body,
  c.name AS conference
FROM `my_proj.sagebase_linked.meetings` AS m
JOIN `my_proj.sagebase_linked.conferences` AS c ON m.conference_id = c.id
JOIN `my_proj.sagebase_linked.governing_bodies` AS gb ON c.governing_body_id = gb.id
WHERE m.date IS NOT NULL
ORDER BY m.date DESC
LIMIT 5
```

### 都道府県別の議事録件数

```sql
SELECT
  gb.prefecture,
  COUNT(DISTINCT mn.id) AS minutes_count
FROM `my_proj.sagebase_linked.minutes` AS mn
JOIN `my_proj.sagebase_linked.meetings` AS m ON mn.meeting_id = m.id
JOIN `my_proj.sagebase_linked.conferences` AS c ON m.conference_id = c.id
JOIN `my_proj.sagebase_linked.governing_bodies` AS gb ON c.governing_body_id = gb.id
WHERE gb.prefecture IS NOT NULL
GROUP BY gb.prefecture
ORDER BY minutes_count DESC
LIMIT 10
```

### 会派ごとの議案賛否件数（会派レベル）

`proposal_vote_records` は「議案 × 判断者 (会派)」単位の投票記録で、`judgment`（賛成/反対/…）と
`group_name` を持ちます。`proposal_vote_parliamentary_groups` は投票記録と会派マスタを結ぶ中間表です。

```sql
SELECT
  pg.name AS parliamentary_group,
  pvr.judgment,
  COUNT(*) AS votes
FROM `my_proj.sagebase_linked.proposal_vote_parliamentary_groups` AS pvg
JOIN `my_proj.sagebase_linked.proposal_vote_records` AS pvr
  ON pvg.judge_id = pvr.id
JOIN `my_proj.sagebase_linked.parliamentary_groups` AS pg
  ON pvg.parliamentary_group_id = pg.id
GROUP BY parliamentary_group, judgment
ORDER BY parliamentary_group, votes DESC
LIMIT 20
```

- 全クエリは `bq query --dry_run` で構文・カラム名の妥当性を検証済みです（sagebase-gcp 側のマスターデータで確認）。
- 追加のクエリ例（GQL / プレーン SQL 併記）は [docs/graph-queries/](docs/graph-queries/README.md) を参照してください。

## ロードマップ

今後のデータ拡充計画は [Milestones](https://github.com/sage-base/sagebase-community/milestones) で公開しています。

| Milestone | 概要 |
|-----------|------|
| [全 47 都道府県議会の会議データ取得](https://github.com/sage-base/sagebase-community/milestone/1) | 全都道府県の議会会議録を取得・公開 |
| [政令指定都市議会の会議データ取得](https://github.com/sage-base/sagebase-community/milestone/2) | 全 20 政令指定都市の議会会議録を取得・公開 |
| [地方議会の議案・賛否データ取得](https://github.com/sage-base/sagebase-community/milestone/3) | 地方議会の議案・記名投票データを取得・公開 |

追加してほしいデータがあれば、[データ追加リクエスト](https://github.com/sage-base/sagebase-community/issues/new?template=data-request.yml)からお知らせください。リクエストは今後の計画に反映されます。

## データに問題を見つけたら

以下の Issue テンプレートからご報告ください:

- [データ修正の報告](../../issues/new?template=data-correction.yml) - 誤ったデータの修正依頼
- [データ追加のリクエスト](../../issues/new?template=data-request.yml) - 新しいデータの追加依頼

詳しい報告方法は [CONTRIBUTING.md](CONTRIBUTING.md) をご覧ください。

## sagebase_id について

sagebase 上の全レコードには `sagebase_id` という一意の識別子が付与されています。データの誤りを報告する際にこの ID を添えていただくと、対象レコードを正確に特定できます。

フォーマット: `{プレフィックス}_{UUID}`

代表例（詳細は [docs/sagebase-id.md](docs/sagebase-id.md)）:

| プレフィックス | エンティティ |
|:---:|---|
| `pol` | 政治家 |
| `pty` | 政党 |
| `gov` | 開催主体（議会） |
| `mtg` | 会議 |
| `mnt` | 議事録 |
| `cvs` | 発言 |
| `prp` | 議案 |

## ライセンスと利用条件

sagebase が公開するデータは、出自の異なる二つの層でできています。

**sagebase が付与した層** — テーブル/ノード/エッジの構造、正規化された名称・日付、名寄せ結果（政治家・会派・会議体の同定）、`sagebase_id`、集計値、`matching_confidence` 等の品質メタデータ、本リポジトリのドキュメント — は **[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ja)** で提供します。出典を表示していただければ、商用・非商用を問わず、複製・再配布・加工・データベースとしての再構築が自由にできます（CC BY 4.0 はデータベース権も明示的にカバーします）。

**議会等が公開した情報の層** — 議事録の発言原文、議案の本文・件名、採決結果の記載など — は各議会の公開条件に由来します。sagebase はこの部分にライセンスを付与する立場になく、原文を大量に再配布する用途（発言テキストのコーパス公開等）では、出典元の議会が定める利用条件をご自身でご確認ください。

### 出典表示の推奨形式

そのまま利用する場合:

```
出典: Sagebase（一般社団法人政治ベース）https://sage-base.com/ — CC BY 4.0
```

加工して利用する場合:

```
一般社団法人政治ベース「Sagebase」（https://sage-base.com/）を加工して作成。
ライセンス: CC BY 4.0（https://creativecommons.org/licenses/by/4.0/deed.ja）
```

学術論文・レポートで引用する場合は、データが随時更新されるため参照日（または BigQuery のスナップショット日時）を添えてください。表示スペースが限られる場合の短縮形を含め、詳しくは[利用規約 第 3 条](https://sage-base.com/terms/#3-出典表示の推奨形式)に記載しています。

### 免責

本データの相当部分は、LLM による抽出・名寄せを含む自動処理を経て作られており、**誤りは残ります**。重要な判断・報道・研究の根拠とする前に、一次情報（各議会の公開する議事録等）と照合してください。

名寄せ結果を含むテーブルには `matching_confidence` / `min_matching_confidence` / `all_verified` などの品質列を用意しています。低信頼のマッチを確定した事実として扱わないよう、これらの列を条件に含めてご利用ください。また、スキーマは用意されていても値の投入が段階的なカラム・テーブルがあります（造反フラグが全行 NULL、提出者エッジが 0 行など）。「該当なし」と「未反映」は区別してご解釈ください。

本データは現状有姿（AS IS）で提供され、正確性・完全性・特定目的への適合性を保証しません。

### 削除・訂正の依頼

- **データの誤り**: [データ修正の報告](../../issues/new?template=data-correction.yml)からご報告ください
- **ご自身に関する情報の削除・訂正**（議事録に由来して氏名等が掲載されている方ご本人・代理人の方）: 件名に **【削除・訂正依頼】** と記載のうえ [info@sage-base.org](mailto:info@sage-base.org) までご連絡ください。判断の考え方・必要な情報・回答の目安期間は[プライバシーポリシー 第 5 条](https://sage-base.com/privacy/#5-削除訂正の依頼)に記載しています。**公開の GitHub Issue ではなくメールでお願いします**
- **権利侵害の申し立て**: [info@sage-base.org](mailto:info@sage-base.org)

正式な条件は[利用規約](https://sage-base.com/terms/)と[プライバシーポリシー](https://sage-base.com/privacy/)をご確認ください。要約は[ライセンスページ](https://sage-base.com/license/)にあります。

## ドキュメント

- [sagebase_id について](docs/sagebase-id.md) - データ識別子の詳細
- [データカバレッジ](docs/data-coverage.md) - 収集しているデータの範囲
- [graph-queries](docs/graph-queries/README.md) - graph 層データマート向け代表クエリ集（GQL + プレーン SQL 併記）

## リンク

- [sage-base.com](https://sage-base.com) - プロダクトサイト
- [BigQuery Analytics Hub コンソール](https://console.cloud.google.com/bigquery/analytics-hub) - リスティング検索・購読
- [利用規約](https://sage-base.com/terms/) / [ライセンス（要約）](https://sage-base.com/license/) / [プライバシーポリシー](https://sage-base.com/privacy/)
