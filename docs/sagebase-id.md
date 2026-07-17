# sagebase_id について

## 概要

sagebase 上の全レコードには **sagebase_id** という一意の識別子が付与されています。データの誤りを報告する際にこの ID を添えていただくと、対象レコードを正確に特定できます。

## フォーマット

```
{プレフィックス}_{UUID}
```

- **プレフィックス**: エンティティ種別を示す 3 文字の短縮形
- **UUID**: UUID v4（ハイフン付き 36 文字）

例: `pol_550e8400-e29b-41d4-a716-446655440000`

## プレフィックス一覧

BigQuery `sagebase-gcp.sagebase` の実データから確認したプレフィックスです。

| プレフィックス | エンティティ | 説明 |
|:---:|---|---|
| `pol` | 政治家 | 国会議員・地方議員等の基本情報 |
| `pty` | 政党 | 政党の基本情報 |
| `elc` | 選挙 | 選挙の基本情報 |
| `elm` | 選挙結果メンバー | 選挙における候補者の結果 |
| `gov` | 開催主体 | 議会等の組織（国会、都道府県議会、市町村議会） |
| `cnf` | 会議体 | 本会議・常任委員会・特別委員会等 |
| `cnm` | 会議体メンバー | 委員会等の所属議員 |
| `mtg` | 会議 | 個別の会議 |
| `mnt` | 議事録 | 会議の議事録 |
| `cvs` | 発言 | 議事録内の個別発言 |
| `spk` | 発言者 | 議事録から抽出された発言者名 |
| `prp` | 議案 | 法案・決議等 |
| `pjd` | 議案賛否（個人） | 議案に対する個人の賛否集計 |
| `psb` | 議案提出者 | 議案の提出者 |
| `ppj` | 議案記名投票 | 個人 × 議案 × 採決の生記録 |
| `jpg` | 議案賛否（会派） | 議案に対する会派単位の判断 |
| `pgr` | 議員団（会派） | 会派の基本情報 |
| `gof` | 政府関係者 | 大臣等の政府関係者 |

## 確認方法

### BigQuery

Analytics Hub でリスティングを購読して自分のプロジェクトに linked dataset を作成した後、各テーブルの `sagebase_id` カラムで確認できます。

```sql
SELECT sagebase_id, name
FROM `my_proj.sagebase_linked.politicians`
WHERE name LIKE '%山田%'
LIMIT 20
```

`my_proj.sagebase_linked` は購読時に作成した自分の linked dataset 名に置き換えてください。

### sage-base.com

今後、各データページに sagebase_id を表示する機能を追加予定です。

## フィードバック時の使い方

sagebase_id がわかる場合は、[データ修正報告フォーム](https://github.com/sage-base/sagebase-community/issues/new?template=data-correction.yml) に記入してください。ID がわからない場合でも、対象データが特定できる情報（政治家名、会議名など）を記載いただければ対応可能です。
