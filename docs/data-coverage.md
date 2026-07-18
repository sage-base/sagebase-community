# データカバレッジ

sagebase が現在収集・公開しているデータの範囲です。カバレッジは順次拡充しています（最新は BigQuery `sagebase-gcp.sagebase` の実データが正）。

## 国会データ

| データ種別 | 衆議院 | 参議院 | ソース |
|-----------|:---:|:---:|------|
| 議事録・発言 | :white_check_mark: | :white_check_mark: | 国会会議録検索システム |
| 議案・法案 | :white_check_mark: | :white_check_mark: | 衆議院/参議院公式サイト |
| 議案賛否（本会議・会派） | :white_check_mark: | :white_check_mark: | 衆議院/参議院公式サイト |
| 議案賛否（本会議・個人記名投票） | 段階拡充中 | :white_check_mark: | 参議院公式サイト |
| 選挙結果 | :white_check_mark: | :white_check_mark: | 総務省選挙関連資料 |
| 政治家情報 | :white_check_mark: | :white_check_mark: | 各政党公式サイト等 |
| 会派（院内会派） | :white_check_mark: | :white_check_mark: | 国会公式サイト |
| 委員会構成 | :white_check_mark: | :white_check_mark: | 国会公式サイト |

## 地方議会データ

### 都道府県議会（47 議会）

**全 47 都道府県議会の議事録・発言データを取得・公開しています**。議会サイトはそれぞれ独自の CMS（kaigiroku.net / dbsr.jp / gijiroku ASP 系 / 独自 CMS 等）を使っているため、順次対応を追加してきました。

議案・議決データについては、対応議会を段階的に追加しています。

### 政令指定都市（20 都市）

**一部の政令指定都市**の議事録・発言データを取得・公開しています。全 20 都市への対応を進めています（[Milestone 2](https://github.com/sage-base/sagebase-community/milestone/2)）。

### 一般市町村議会

順次拡充中です。追加してほしい議会があれば、[データ追加リクエスト](https://github.com/sage-base/sagebase-community/issues/new?template=data-request.yml)からお知らせください。

## 実データ規模（目安）

| データ | 件数 |
|---|---:|
| 開催主体（governing_bodies） | 1,795 |
| 会議体（conferences） | 4,533 |
| 会議（meetings） | 27 万件超 |
| 議事録（minutes） | 31 万件超 |
| 発言（conversations） | 2,200 万件超 |
| 政治家（politicians） | 19,187 |
| 政党（political_parties） | 139 |
| 議員団・会派（parliamentary_groups） | 375 |
| 議案（proposals） | 12.6 万件超 |
| 議案賛否（proposal_judges・個人集計） | 13.2 万件超 |
| 議案賛否（proposal_vote_parliamentary_groups・会派単位） | 1.6 万件超 |
| 議案 × 判断者単位の投票記録（proposal_vote_records） | 197 万件超 |
| 選挙（elections） | 3.4 万件超 |

（数値は 2026-07 時点の目安。最新は BQ `__TABLES__` の row_count が正。）

## 公開データセット

BigQuery Analytics Hub 経由で公開しています。購読手順は [README#購読手順](../README.md#購読手順bigquery-console) を参照。

| データセット | 種類 | 内容 |
|-------------|:---:|------|
| `sagebase-gcp.sagebase` | main 層 | 23 テーブル。安定版。 |
| `sagebase-gcp.sagebase_graph` | graph 層（Preview） | ノード 8 / エッジ 17 の graph-ready 形式。 |

**Analytics Hub 経由でしか読めません**。両データセット直接は allAuthenticatedUsers に開放していないため、リスティング（`sagebase_data` / `sagebase_graph_data`）を購読して自分のプロジェクトに linked dataset を作成してから参照します。

## データの更新頻度

- 議事録: 各議会サイトへの掲載後に取得（通常、会議から数日〜数週間後）
- 議案・賛否: 定期的にスクレイピング
- 選挙結果: 選挙実施後に順次追加
- BigQuery: パイプライン実行時（当日中に反映）
