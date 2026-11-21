# Overture Maps データ構造（release 2026-11-18.0）

公開S3 `s3://overturemaps-us-west-2/release/2026-11-18.0/` の Parquet から DESCRIBE で取得したスキーマ。

## 全体構成

```
release/2026-11-18.0/theme=<テーマ>/type=<タイプ>/*.parquet   ← Hive形式の分割、GeoParquet
```

| theme | type | 内容 |
|---|---|---|
| addresses | address | 住所点 |
| base | land / water / land_use / land_cover / infrastructure / bathymetry | 背景地図用（陸地、水域、土地利用、被覆、構造物、水深） |
| buildings | building / building_part | 建物と、その部位（3D表現用） |
| divisions | division / division_area / division_boundary | 行政区画（代表点 / 面 / 境界線） |
| places | place | POI |
| transportation | segment / connector | 道路・鉄道・水路ネットワーク（線 / 結節点） |

## 全タイプ共通の列

- `id`：GERS ID。リリースをまたいで同じ地物を指す安定ID
- `geometry`：WKB（BLOB）。DuckDB では `ST_GeomFromWKB`
- `bbox`：`STRUCT(xmin, xmax, ymin, ymax)`。範囲での絞り込みはこの列で行うと、Parquet統計で読むファイルが減って速い
- `sources[]`：出典（dataset, license, record_id, update_time, confidence など）
- `version`：地物の版番号

## 多くのタイプにある列

- `names`：`primary`（主名称）、`common`（言語コード→名称のMAP。例 `names.common['ja']`）、`rules[]`（別名・略称・国ごとの見え方など）
- `subtype` / `class`：主な分類の2段階（transportation は `subclass` も）
- `level`（Z順序）、`wikidata`、`source_tags`（元データ、主にOSMのタグのMAP）

## タイプごとの特徴

### transportation/segment
- `connectors[]`：`{connector_id, at}`。`at` は線上の位置（0〜1）で、トポロジーはこの列で組む
- `speed_limits` / `access_restrictions` / `road_flags` / `width_rules` / `level_rules` / `road_surface`：`between: [開始, 終了]` による線形参照。線分のどの区間に当てはまる属性かを表し、1本の線に1つの値とは限らない
- `prohibited_transitions`（右左折禁止）、`routes`（路線番号）、`destinations`（方面案内）

### places/place
- `taxonomy {primary, hierarchy[], alternates[]}` と `basic_category`（以前の `categories` 列は無くなっている）
- `confidence`、`operating_status`、`brand`、`addresses[]`、`websites` / `phones` / `socials` / `emails`

### buildings
- `height`、`num_floors`、`roof_shape`、`roof_material`、`facade_color` など3D用属性
- `has_parts` が true の建物は building_part を持ち、`building_part.building_id` で元の建物とつながる

### divisions
- `division`（点）が本体。`division_area.division_id` と `division_boundary.division_ids[]` がこれを参照
- `hierarchies`（上位区画の並び）、`parent_division_id`、`admin_level`、`population`、`perspectives`（係争地をどの国の立場で表すか）

### addresses
- `street`、`number`、`unit`、`postcode`、`address_levels[]`（上位の地名）、`country`

## DuckDB での読み方の例

```sql
INSTALL spatial; LOAD spatial; INSTALL httpfs; LOAD httpfs; SET s3_region='us-west-2';
SELECT id, names.primary, taxonomy.primary, geometry
FROM read_parquet('s3://overturemaps-us-west-2/release/2026-11-18.0/theme=places/type=place/*')
WHERE bbox.xmin BETWEEN 141.30 AND 141.40 AND bbox.ymin BETWEEN 43.03 AND 43.10;  -- 札幌中心部
```

## 全列一覧
### addresses/address

```
id : VARCHAR
street : VARCHAR
number : VARCHAR
unit : VARCHAR
postcode : VARCHAR
postal_city : VARCHAR
address_levels : STRUCT("value" VARCHAR)[]
country : VARCHAR
geometry : BLOB
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/bathymetry

```
id : VARCHAR
depth : INTEGER
cartography : STRUCT(prominence INTEGER, min_zoom INTEGER, max_zoom INTEGER, sort_key INTEGER)
geometry : BLOB
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/infrastructure

```
id : VARCHAR
geometry : BLOB
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
level : INTEGER
wikidata : VARCHAR
source_tags : MAP(VARCHAR, VARCHAR)
subtype : VARCHAR
class : VARCHAR
height : DOUBLE
surface : VARCHAR
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/land

```
id : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
subtype : VARCHAR
class : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
source_tags : MAP(VARCHAR, VARCHAR)
level : INTEGER
wikidata : VARCHAR
surface : VARCHAR
elevation : INTEGER
geometry : BLOB
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/land_cover

```
id : VARCHAR
subtype : VARCHAR
cartography : STRUCT(prominence INTEGER, min_zoom INTEGER, max_zoom INTEGER, sort_key INTEGER)
geometry : BLOB
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/land_use

```
id : VARCHAR
geometry : BLOB
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
level : INTEGER
wikidata : VARCHAR
source_tags : MAP(VARCHAR, VARCHAR)
subtype : VARCHAR
class : VARCHAR
elevation : INTEGER
surface : VARCHAR
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### base/water

```
id : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
subtype : VARCHAR
class : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
source_tags : MAP(VARCHAR, VARCHAR)
level : INTEGER
wikidata : VARCHAR
is_intermittent : BOOLEAN
is_salt : BOOLEAN
geometry : BLOB
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### buildings/building

```
id : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
level : INTEGER
height : DOUBLE
min_height : DOUBLE
is_underground : BOOLEAN
num_floors : INTEGER
num_floors_underground : INTEGER
min_floor : INTEGER
subtype : VARCHAR
class : VARCHAR
facade_color : VARCHAR
facade_material : VARCHAR
roof_material : VARCHAR
roof_shape : VARCHAR
roof_direction : DOUBLE
roof_orientation : VARCHAR
roof_color : VARCHAR
roof_height : DOUBLE
geometry : BLOB
has_parts : BOOLEAN
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### buildings/building_part

```
id : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
height : DOUBLE
min_height : DOUBLE
is_underground : BOOLEAN
num_floors : INTEGER
num_floors_underground : INTEGER
min_floor : INTEGER
facade_color : VARCHAR
facade_material : VARCHAR
roof_material : VARCHAR
roof_shape : VARCHAR
roof_direction : DOUBLE
roof_orientation : VARCHAR
roof_color : VARCHAR
roof_height : DOUBLE
level : INTEGER
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
geometry : BLOB
building_id : VARCHAR
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### divisions/division

```
id : VARCHAR
geometry : BLOB
country : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
subtype : VARCHAR
admin_level : INTEGER
class : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
wikidata : VARCHAR
perspectives : STRUCT("mode" VARCHAR, countries VARCHAR[])
local_type : MAP(VARCHAR, VARCHAR)
region : VARCHAR
hierarchies : STRUCT(division_id VARCHAR, subtype VARCHAR, "name" VARCHAR)[][]
parent_division_id : VARCHAR
norms : STRUCT(driving_side VARCHAR)
population : INTEGER
capital_division_ids : VARCHAR[]
capital_of_divisions : STRUCT(division_id VARCHAR, subtype VARCHAR)[]
cartography : STRUCT(prominence INTEGER, min_zoom INTEGER, max_zoom INTEGER, sort_key INTEGER)
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### divisions/division_area

```
id : VARCHAR
geometry : BLOB
country : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
subtype : VARCHAR
admin_level : INTEGER
class : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
is_land : BOOLEAN
is_territorial : BOOLEAN
region : VARCHAR
division_id : VARCHAR
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### divisions/division_boundary

```
id : VARCHAR
geometry : BLOB
division_ids : VARCHAR[]
subtype : VARCHAR
admin_level : INTEGER
class : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
perspectives : STRUCT("mode" VARCHAR, countries VARCHAR[])
is_disputed : BOOLEAN
is_land : BOOLEAN
is_territorial : BOOLEAN
country : VARCHAR
region : VARCHAR
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### places/place

```
id : VARCHAR
geometry : BLOB
confidence : DOUBLE
websites : VARCHAR[]
emails : VARCHAR[]
socials : VARCHAR[]
phones : VARCHAR[]
brand : STRUCT(wikidata VARCHAR, "names" STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[]))
addresses : STRUCT(freeform VARCHAR, locality VARCHAR, postcode VARCHAR, region VARCHAR, country VARCHAR)[]
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
operating_status : VARCHAR
basic_category : VARCHAR
taxonomy : STRUCT("primary" VARCHAR, hierarchy VARCHAR[], alternates VARCHAR[])
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### transportation/connector

```
id : VARCHAR
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
geometry : BLOB
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```

### transportation/segment

```
id : VARCHAR
names : STRUCT("primary" VARCHAR, common MAP(VARCHAR, VARCHAR), rules STRUCT(variant VARCHAR, "language" VARCHAR, perspectives STRUCT("mode" VARCHAR, countries VARCHAR[]), "value" VARCHAR, "between" DOUBLE[], side VARCHAR)[])
subtype : VARCHAR
class : VARCHAR
subclass : VARCHAR
subclass_rules : STRUCT("value" VARCHAR, "between" DOUBLE[])[]
connectors : STRUCT(connector_id VARCHAR, "at" DOUBLE)[]
road_surface : STRUCT("value" VARCHAR, "between" DOUBLE[])[]
road_flags : STRUCT("values" VARCHAR[], "between" DOUBLE[])[]
rail_flags : STRUCT("values" VARCHAR[], "between" DOUBLE[])[]
width_rules : STRUCT("value" DOUBLE, "between" DOUBLE[])[]
level_rules : STRUCT("value" INTEGER, "between" DOUBLE[])[]
access_restrictions : STRUCT(access_type VARCHAR, "when" STRUCT(during VARCHAR, heading VARCHAR, "using" VARCHAR[], recognized VARCHAR[], "mode" VARCHAR[], vehicle STRUCT(dimension VARCHAR, comparison VARCHAR, "value" DOUBLE, unit VARCHAR)[]), "between" DOUBLE[])[]
speed_limits : STRUCT(min_speed STRUCT("value" INTEGER, unit VARCHAR), max_speed STRUCT("value" INTEGER, unit VARCHAR), is_max_speed_variable BOOLEAN, "when" STRUCT(during VARCHAR, heading VARCHAR, "using" VARCHAR[], recognized VARCHAR[], "mode" VARCHAR[], vehicle STRUCT(dimension VARCHAR, comparison VARCHAR, "value" DOUBLE, unit VARCHAR)[]), "between" DOUBLE[])[]
prohibited_transitions : STRUCT("sequence" STRUCT(connector_id VARCHAR, segment_id VARCHAR)[], final_heading VARCHAR, "when" STRUCT(heading VARCHAR, during VARCHAR, "using" VARCHAR[], recognized VARCHAR[], "mode" VARCHAR[], vehicle STRUCT(dimension VARCHAR, comparison VARCHAR, "value" DOUBLE, unit VARCHAR)[]), "between" DOUBLE[])[]
routes : STRUCT("name" VARCHAR, network VARCHAR, "ref" VARCHAR, symbol VARCHAR, wikidata VARCHAR, "between" DOUBLE[])[]
destinations : STRUCT(labels STRUCT("value" VARCHAR, "type" VARCHAR)[], symbols VARCHAR[], from_connector_id VARCHAR, to_segment_id VARCHAR, to_connector_id VARCHAR, "when" STRUCT(heading VARCHAR), final_heading VARCHAR)[]
sources : STRUCT(property VARCHAR, dataset VARCHAR, license VARCHAR, record_id VARCHAR, update_time VARCHAR, confidence DOUBLE, "between" DOUBLE[], provider VARCHAR, resource VARCHAR, "version" VARCHAR)[]
geometry : BLOB
version : INTEGER
bbox : STRUCT(xmin DOUBLE, xmax DOUBLE, ymin DOUBLE, ymax DOUBLE)
```
