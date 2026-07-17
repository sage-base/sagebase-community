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

```sql
SELECT
  pg.name AS parliamentary_group,
  pvg.judgment,
  COUNT(*) AS votes
FROM `my_proj.sagebase_linked.proposal_vote_parliamentary_groups` AS pvg
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

## ドキュメント

- [sagebase_id について](docs/sagebase-id.md) - データ識別子の詳細
- [データカバレッジ](docs/data-coverage.md) - 収集しているデータの範囲
- [graph-queries](docs/graph-queries/README.md) - graph 層データマート向け代表クエリ集（GQL + プレーン SQL 併記）

## リンク

- [sage-base.com](https://sage-base.com) - プロダクトサイト
- [BigQuery Analytics Hub コンソール](https://console.cloud.google.com/bigquery/analytics-hub) - リスティング検索・購読
