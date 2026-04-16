# sagebase community

政治活動データの修正・追加リクエストを受け付けるコミュニティリポジトリです。

## sagebase とは

[sagebase](https://sage-base.com) は、日本の政治活動データ（国会議事録・議案・賛否・選挙結果など）を収集・構造化し、誰でもアクセスできる形で公開するプロジェクトです。

### 収集しているデータ

| データ種別 | 対象 | ソース |
|-----------|------|--------|
| 議事録・発言 | 衆議院・参議院 | 国会会議録検索システム |
| 議案・法案 | 衆議院・参議院 | 衆議院/参議院公式サイト |
| 議案賛否 | 衆議院・参議院 | 衆議院/参議院公式サイト |
| 選挙結果 | 衆院選・参院選 | 総務省選挙関連資料 |
| 政治家情報 | 国会議員 | 各政党公式サイト等 |
| 会派情報 | 衆議院・参議院 | 国会公式サイト |
| 委員会構成 | 衆議院・参議院 | 国会公式サイト |
| 議事録・発言 | 京都市議会・豊橋市議会 | kaigiroku.net |

### 公開データセット

BigQuery 上で公開データセットとして利用できます:
- `sagebase-gcp.sagebase` (メインビュー)
- `sagebase-gcp.sagebase_raw_vault` (Data Vault)
- `sagebase-gcp.sagebase_mart` (分析用マート)

## ロードマップ

今後のデータ拡充計画は [Milestones](https://github.com/sage-base/sagebase-community/milestones) で公開しています。

| Milestone | 概要 |
|-----------|------|
| [全47都道府県議会の会議データ取得](https://github.com/sage-base/sagebase-community/milestone/1) | 全都道府県の議会会議録を取得・公開 |
| [政令指定都市議会の会議データ取得](https://github.com/sage-base/sagebase-community/milestone/2) | 全20政令指定都市の議会会議録を取得・公開 |
| [地方議会の議案・賛否データ取得](https://github.com/sage-base/sagebase-community/milestone/3) | 地方議会の議案・記名投票データを取得・公開 |

追加してほしいデータがあれば、[データ追加リクエスト](https://github.com/sage-base/sagebase-community/issues/new?template=data-request.yml)からお知らせください。リクエストは今後の計画に反映されます。

## データに問題を見つけたら

以下のIssueテンプレートからご報告ください:

- [データ修正の報告](../../issues/new?template=data-correction.yml) - 誤ったデータの修正依頼
- [データ追加のリクエスト](../../issues/new?template=data-request.yml) - 新しいデータの追加依頼

詳しい報告方法は [CONTRIBUTING.md](CONTRIBUTING.md) をご覧ください。

## sagebase_id について

sagebase上の全レコードには `sagebase_id` という一意の識別子が付与されています。データの誤りを報告する際にこのIDを添えていただくと、対象レコードを正確に特定できます。

フォーマット: `{プレフィックス}_{UUID}`

| プレフィックス | エンティティ | 例 |
|-------------|-----------|-----|
| `pol` | 政治家 | `pol_550e8400-e29b-41d4-a716-446655440000` |
| `pty` | 政党 | `pty_...` |
| `gov` | 議会 | `gov_...` |
| `elc` | 選挙 | `elc_...` |
| `prp` | 議案 | `prp_...` |
| `spk` | 発言者 | `spk_...` |
| `cnv` | 発言 | `cnv_...` |

## ドキュメント

- [sagebase_id について](docs/sagebase-id.md) - データ識別子の詳細
- [データカバレッジ](docs/data-coverage.md) - 収集しているデータの範囲

## リンク

- [sage-base.com](https://sage-base.com) - プロダクトサイト
- [BigQuery 公開データセット](https://console.cloud.google.com/bigquery?project=sagebase-gcp) - データセット
