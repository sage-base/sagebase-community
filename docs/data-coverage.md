# データカバレッジ

sagebase が現在収集・公開しているデータの範囲です。

## 国会データ

| データ種別 | 衆議院 | 参議院 | ソース |
|-----------|:---:|:---:|------|
| 議事録・発言 | :white_check_mark: | :white_check_mark: | 国会会議録検索システム |
| 議案・法案 | :white_check_mark: | :white_check_mark: | 衆議院/参議院公式サイト |
| 議案賛否（本会議） | :white_check_mark: | :white_check_mark: | 衆議院/参議院公式サイト |
| 選挙結果 | :white_check_mark: | :white_check_mark: | 総務省選挙関連資料 |
| 政治家情報 | :white_check_mark: | :white_check_mark: | 各政党公式サイト等 |
| 会派（院内会派） | :white_check_mark: | :white_check_mark: | 国会公式サイト |
| 委員会構成 | :white_check_mark: | :white_check_mark: | 国会公式サイト |

## 地方議会データ

現在、地方議会データの収集を準備中です。

| 対象 | 状態 | 備考 |
|------|------|------|
| 京都市議会 | 準備中 | 国会用スキーマの地方議会汎用性検証として着手予定 |

地方議会のデータを追加してほしい場合は、[データ追加リクエスト](https://github.com/sage-base/sagebase-community/issues/new?template=data-request.yml)からリクエストをお送りください。

## 公開データセット

BigQuery 上で以下のデータセットを公開しています。

| データセット | 説明 |
|-------------|------|
| `sage-base-sagebase.sagebase` | メインビュー（最新のデータ） |
| `sage-base-sagebase.sagebase_raw_vault` | Data Vault（変更履歴を含む） |
| `sage-base-sagebase.sagebase_mart` | 分析用マート（集計・加工済みビュー） |

## データの更新頻度

- 議事録: 国会会議録検索システムへの掲載後に取得（通常、会議から数日〜数週間後）
- 議案・賛否: 定期的にスクレイピング
- 選挙結果: 選挙実施後に手動で追加
- BigQuery: パイプライン実行時に更新
