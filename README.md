# overture-maps-jp-validation

[Overture Maps](https://overturemaps.org/) のデータについて、日本国内での**網羅性**と**正確性**を検証するためのリポジトリです。政府の公的統計や公的な空間データと比較して評価します。

> [!NOTE]
> 本リポジトリは Overture Maps Foundation の公式プロジェクトではありません。

## 対象

- Overture Maps release: `2026-11-18.0`
- 範囲: 日本
- テーマ: divisions / buildings / places / transportation / addresses / base

## 方針

詳しくは [docs/methodology.md](docs/methodology.md) を参照してください。

1. **棚卸し**: 日本分を切り出し、テーマ別の件数と出典（`sources.dataset`）の構成を把握する
2. **統計との比較（広く浅く）**: 市区町村やメッシュの単位で、公的統計との件数比・延長比などを全国について出す
3. **地物単位の照合（狭く深く）**: パイロット地域で公的な空間データと1件ずつ照合し、漏れ（欠落）と余分（存在しない・重複した地物）を分けて測る
4. **判断**: 用途ごとに合格ラインを決めて評価する

データの構造は [docs/schema.md](docs/schema.md) にまとめています。

## 構成

```
docs/        検証方針、スキーマのまとめ
sql/         DuckDB のクエリ
src/         照合・集計の処理（Python）
data/        取得したデータ（git では管理しない）
results/     集計表と地図
```

## セットアップ

[uv](https://docs.astral.sh/uv/) を使います。

```sh
uv sync
```

## ライセンス

コードは [MIT License](LICENSE) で公開しています。

元データにはそれぞれのライセンスが適用されます。

- **Overture Maps**: テーマごとにライセンスが異なります。OpenStreetMap 由来の部分は ODbL です。詳しくは [Overture のドキュメント](https://docs.overturemaps.org/attribution/) を参照してください。
- **公的統計・公的な空間データ**: 各提供元の利用規約に従ってください。
