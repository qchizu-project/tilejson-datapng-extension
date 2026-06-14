# TileJSON DataPNG Extension (Draft)

**バージョン: 0.5.0 (2026-06-14)**

データPNG仕様（[データPNG](https://gsj-seamless.jp/labs/datapng/)）に基づくタイルセットのメタデータを TileJSON 3.0.0 に記述するための拡張仕様（案）。

> **規範語の定義**: 本仕様において MUST / MUST NOT / SHOULD / MAY は [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) に従う。

---

## 1. 概要

TileJSON 3.0.0 は地図タイルセットの汎用メタデータ規格だが、**データPNG**（数値PNG・パレットPNG）をタイルとして配信する場合に必要な属性情報を記述する手段を持たない。本拡張は、TileJSON の仕様（未知キーを無視する）を利用し、`datapng` キーを追加することでクライアントがタイルのデコード・描画に必要な情報を事前に取得できるようにする。

### 1.1 設計方針

- TileJSON 3.0.0 との**後方互換性**を維持する（未知キーとして無視可能）。
- データPNG仕様（グリッドPNGタイル仕様 v0.1）に準拠する。
- 本バージョンでは**グリッドPNG**（数値PNG・パレットPNG）を対象とする。
- 測量年次・データソース・鉛直基準面（測地系）等の自由記載は TileJSON 既存の `description` フィールドを使用する。

### 1.2 対象外（将来拡張）

以下は本バージョンの対象外とし、将来バージョンで検討する。

- リストPNG（List PNG。固定長レコードデータ。点群PNG（Point Cloud PNG）を含む）
- Mapbox Terrain-RGB 等の外部エンコーディング互換

### 1.3 参照仕様

| 仕様 | URL |
|------|-----|
| TileJSON 3.0.0 | https://github.com/mapbox/tilejson-spec/tree/master/3.0.0 |
| RFC 2119 | https://datatracker.ietf.org/doc/html/rfc2119 |
| データPNG | https://gsj-seamless.jp/labs/datapng/ |
| グリッドPNGタイル仕様 | https://gsj-seamless.jp/labs/datapng/gridpngtileSpec.html |

### 1.4 バージョニング

本仕様は [Semantic Versioning 2.0.0](https://semver.org/) に従う。0.x.y の間は後方互換性を保証しない。1.0.0 への移行は、複数の独立した実装による相互運用性の確認をもって行う。`datapng` オブジェクト内にバージョン番号フィールドは設けない。

---

## 2. 拡張キー `datapng`

TileJSON のルートオブジェクトに **`datapng`** キー（Object）を追加する。

```jsonc
{
  "tilejson": "3.0.0",
  "tiles": [ "https://tiles.gsj.jp/tiles/elev/mixed/{z}/{y}/{x}.png" ],
  "minzoom": 0,
  "maxzoom": 15,
  "tileSize": 256,

  "datapng": {
    // ── 本拡張で定義するフィールド群 ──
  }
}
```

### 2.1 `tileSize` — タイルサイズ

TileJSON 3.0.0 にはラスタータイルのピクセルサイズを示すフィールドがない。本拡張では TileJSON のルートオブジェクトに **`tileSize`** フィールド（Number）を追加する。

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `tileSize` | Number | **REQUIRED** | タイル画像の一辺のピクセル数（例: `256`, `512`） |

> **注記**: `tileSize` はタイル画像の物理的なピクセルサイズであり、データソース固有の属性である。MapLibre GL JS 等の地図ライブラリでは Style JSON のソース定義で `tileSize` を指定するが、同一ソースを複数レイヤーで共有する場合、データソース側（TileJSON）で定義する方が合理的である。

---

## 3. フィールド定義

クライアントは `datapng` オブジェクト内の未知のキーを無視しなければならない（MUST）。

`datapng.type` に対して該当しないフィールド（例: `type: "palette"` 時の `factor`・`offset`）が含まれていた場合、クライアントはそれらを無視しなければならない（MUST）。

### 3.1 `datapng.type` — データPNG種別

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `type` | String (enum) | **REQUIRED** | データPNGの種類 |

指定可能な値:

| 値 | 意味 |
|----|------|
| `"numerical"` | 数値PNG（Numerical PNG） |
| `"palette"` | パレットPNG（Palette PNG） |

```json
{ "datapng": { "type": "numerical" } }
```

### 3.2 数値PNG用フィールド

`type` が `"numerical"` の場合に使用するフィールド群。

| キー | 型 | 必須 | デフォルト | 説明 |
|------|----|------|-----------|------|
| `factor` | Number | OPTIONAL | `1` | 係数 *f*。 `v = f × rawValue + offset` |
| `offset` | Number | OPTIONAL | `0` | オフセット *o* |
| `unit` | String | OPTIONAL | — | 変換後の値の単位（例: `"m"`, `"cm"`, `"℃"`） |
| `invalidColor` | Array[3] of int | OPTIONAL | — | 追加無効色 `[r, g, b]`。透明ピクセルに加えて無効値として扱う色（1色のみ指定可能） |
| `dataRange` | Object | OPTIONAL | — | デコード後の値の期待範囲。`min`（Number）と `max`（Number）を持つ |
| `precision` | Number | OPTIONAL | — | 元データの有効な最小単位。`factor` はエンコーディングの分解能（例: 0.01m刻み）であり、`precision` はデータとして意味のある最小の差（例: 0.1m）を示す |

変換式:

```
r' = (r < 128) ? r : r - 256
rawValue = r' × 65536 + g × 256 + b
v = factor × rawValue + offset
```

`rawValue` は24ビット符号付き整数の全域（-8,388,608 〜 8,388,607）を取りうる。実装は少なくとも32ビット整数型で保持しなければならない（MUST）。

#### 無効値の判定

クライアントは以下の順序で無効値判定を行わなければならない（MUST）:

1. **透明度チェック**: アルファ値が 0 のピクセルは無効値とする。半透明ピクセル（0 < A < 255）は有効として扱う。
2. **invalidColor チェック**: `invalidColor` が指定されている場合、ピクセルの RGB 値が `invalidColor` と完全一致するかを判定する。アルファ値は考慮しない。一致した場合は無効値とする。
3. 上記いずれにも該当しないピクセルに対してのみ、変換式を適用する。

> **補足**: `invalidColor` は1色のみ指定可能とする。

**例: 国土地理院標高タイル（rawValue をメートル単位に変換）**

```json
{
  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "invalidColor": [128, 0, 0],
    "dataRange": { "min": -500, "max": 9000 },
    "precision": 1
  }
}
```

### 3.3 パレットPNG用フィールド

`type` が `"palette"` の場合に使用するフィールド群。

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `legend` | Object or String | **REQUIRED** | 凡例情報（インラインまたはURL参照） |

#### 3.3.1 凡例のインライン定義

JSON凡例フォーマットに準拠した構造をそのまま埋め込む。

**凡例オブジェクト**のフィールド:

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `title` | String | OPTIONAL | 凡例全体のタイトル |
| `items` | Array | **REQUIRED** | 凡例項目オブジェクトの配列 |

**凡例項目オブジェクト**のフィールド:

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `r` | Integer (0–255) | **REQUIRED** | 色の R 値 |
| `g` | Integer (0–255) | **REQUIRED** | 色の G 値 |
| `b` | Integer (0–255) | **REQUIRED** | 色の B 値 |
| `title` | String | **REQUIRED** | 凡例項目の短いタイトル |
| `description` | String | OPTIONAL | 凡例項目の詳細な説明文。プレーンテキストまたはHTMLフラグメント。注釈、出典、適用条件等の補足情報を記載できる |

> **仕様上の拡張性**: JSON凡例フォーマットに従い、凡例項目オブジェクトに上記以外の任意のメンバーを追加することができる。クライアントは処理できないメンバーを無視しなければならない（MUST）。これにより、シンボル画像URL、数値範囲、表示順序等をアプリケーション固有に追加できる。

凡例によるカラーマッチングは、ピクセルの RGB 値と凡例項目の `(r, g, b)` の**完全一致**で行う（MUST）。一致する凡例項目がない場合の描画はクライアント実装依存とする。

```json
{
  "datapng": {
    "type": "palette",
    "legend": {
      "title": "洪水浸水想定区域（想定最大規模）浸水深",
      "items": [
        {
          "r": 245, "g": 245, "b": 50,
          "title": "0.5m未満",
          "description": "床下浸水相当。避難行動は徒歩で可能。"
        },
        {
          "r": 255, "g": 216, "b": 0,
          "title": "0.5～3.0m",
          "description": "1階床上浸水～1階水没相当。水平避難が必要。"
        },
        {
          "r": 255, "g": 153, "b": 0,
          "title": "3.0～5.0m",
          "description": "2階水没相当。垂直避難では不十分な場合がある。"
        },
        {
          "r": 255, "g": 40, "b": 0,
          "title": "5.0～10.0m"
        },
        {
          "r": 165, "g": 0, "b": 33,
          "title": "10.0～20.0m",
          "description": "3階以上の建物も水没する深さ。該当地域では早期の広域避難が不可欠。"
        }
      ]
    }
  }
}
```

#### 3.3.2 凡例の外部参照

凡例データが大きい場合はURLで参照する。`legend` の値が文字列の場合、クライアントはそのURLから JSON凡例フォーマットを取得する（MUST）。フェッチが失敗した場合（HTTPエラー・タイムアウト・CORSエラー等）、クライアントはタイルを PNG 画像としてそのまま表示するか、エラーを上位に通知すべきである（SHOULD）。リトライポリシーはクライアント実装依存とする。

```json
{
  "datapng": {
    "type": "palette",
    "legend": "https://gbank.gsj.jp/seamless/v2/api/1.2/legend.json"
  }
}
```

> **鉛直基準面（測地系）について**: 標高タイル等、値が鉛直方向の物理量を表す場合の基準面（測地系・ジオイドモデル等）は、専用フィールドを設けず TileJSON 既存の `description` フィールドに自由記述する（§1.1）。鉛直基準面は国・地域ごとに異なり、同一地点でも基準面の違いにより標高値が異なるため、データが依拠する基準面を `description` に明示することが望ましい。

### 3.4 `datapng.support` — ピクセル値の support（格納値が代表する領域）

格納されたピクセル値が、セル内のどの幾何学的領域を代表するかを示す。これは地理統計でいう **support**（値の台）であり、値が**点**に対応するか（点 support）、**セル全体**に対応するか（面 support, block support）を区別する。`support` はピクセル値の空間的意味を定義するメタデータであり、クライアントへの動作指示ではない。

| キー | 型 | 必須 | デフォルト | 説明 |
|------|----|------|-----------|------|
| `support` | Object | OPTIONAL | — | 格納値の support（点 / 面）。省略時の解釈はクライアント実装依存 |

**`support` オブジェクトのフィールド:**

| サブキー | 型 | 必須 | 説明 |
|---------|----|------|------|
| `type` | String (enum) | **REQUIRED** | support の種別。`"point"`（点 support）または `"block"`（面 support） |
| `anchor` | String (enum) | OPTIONAL | 値が留まるセル内の点。`type` が `"point"` の場合のみ有効。`"northwest"`（北西端＝左上）または `"center"`（中央） |

| `type` の値 | 意味 |
|------|------|
| `"point"` | 値はセル内の特定の点（`anchor` で指定）に対応する点 support |
| `"block"` | 値は特定の点ではなくセル範囲全体に対応する代表値（面 support / block support） |

`type` が `"block"` の場合、`anchor` が含まれていてもクライアントはこれを無視しなければならない（MUST）。

> **量の性質と support は別物**: `support` は値が代表する幾何学的領域を示すものであり、対象量が本来「点で定義できる量」か否かとは独立である。標高のような内包的（点で定義できる）量であっても、タイル生成時にセル平均で格納されている場合は面 support（`block`）となる（change of support）。`block` は「セル全体の代表値」であることのみを表し、その代表値が平均・最大・合計のいずれで導出されたかは規定しない。集約方法を明示する必要がある場合は、TileJSON 既存の `description` フィールドに自由記述する（§1.1）。

---

## 4. 完全な TileJSON 例

### 4.1 産総研シームレス標高タイル（統合DEM）

```json
{
  "tilejson": "3.0.0",
  "name": "シームレス標高タイル 統合DEM",
  "description": "産総研が提供する統合標高データ。基盤地図情報数値標高モデル(DEM5A等)、ASTER GDEM、GEBCO等を統合。ピクセルは北西端の標高値をcm精度で保持。測地系: JGD2011。",
  "version": "1.1.9",
  "attribution": "産総研地質調査総合センター CC BY 4.0 互換",
  "scheme": "xyz",
  "tiles": [
    "https://tiles.gsj.jp/tiles/elev/mixed/{z}/{y}/{x}.png"
  ],
  "minzoom": 0,
  "maxzoom": 15,
  "bounds": [120.0, 20.0, 155.0, 50.0],

  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "support": { "type": "point", "anchor": "northwest" },
    "dataRange": { "min": -500, "max": 9000 }
  }
}
```

### 4.2 国土地理院 標高タイル PNG形式

```json
{
  "tilejson": "3.0.0",
  "name": "地理院標高タイル（DEM5C）",
  "description": "国土地理院 基盤地図情報数値標高モデル 5mメッシュ。標高基準面: 東京湾平均海面。",
  "attribution": "国土地理院",
  "tiles": [
    "https://cyberjapandata.gsi.go.jp/xyz/dem5c_png/{z}/{x}/{y}.png"
  ],
  "minzoom": 1,
  "maxzoom": 15,

  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "invalidColor": [128, 0, 0],
    "support": { "type": "block" },
    "dataRange": { "min": -500, "max": 9000 },
    "precision": 0.1
  }
}
```

### 4.3 国土交通省 洪水浸水想定区域（パレットPNG）

```json
{
  "tilejson": "3.0.0",
  "name": "洪水浸水想定区域（想定最大規模）",
  "description": "国土交通省ハザードマップポータルサイトより配信。想定し得る最大規模の降雨による浸水深。",
  "attribution": "国土交通省ハザードマップポータルサイト",
  "tiles": [
    "https://disaportaldata.gsi.go.jp/raster/01_flood_l2_shinsuishin_data/{z}/{x}/{y}.png"
  ],
  "minzoom": 2,
  "maxzoom": 17,

  "datapng": {
    "type": "palette",
    "legend": {
      "title": "浸水深",
      "items": [
        { "r": 245, "g": 245, "b": 50, "title": "0.5m未満"    },
        { "r": 255, "g": 216, "b":  0, "title": "0.5～3.0m"   },
        { "r": 255, "g": 153, "b":  0, "title": "3.0～5.0m"   },
        { "r": 255, "g":  40, "b":  0, "title": "5.0～10.0m"  },
        { "r": 165, "g":   0, "b": 33, "title": "10.0～20.0m" }
      ]
    }
  }
}
```

### 4.4 20万分の1日本シームレス地質図（パレットPNG + 外部凡例）

```json
{
  "tilejson": "3.0.0",
  "name": "20万分の1日本シームレス地質図 V2",
  "description": "産総研地質調査総合センター。地質単元ごとにユニークな色が割り当てられたパレットPNGタイル。",
  "attribution": "産総研地質調査総合センター CC BY 4.0 互換",
  "tiles": [
    "https://gbank.gsj.jp/seamless/v2/api/1.2/tiles/{z}/{y}/{x}.png"
  ],
  "minzoom": 0,
  "maxzoom": 13,

  "datapng": {
    "type": "palette",
    "legend": "https://gbank.gsj.jp/seamless/v2/api/1.2/legend.json"
  }
}
```

### 4.5 Copernicus DEM GLO-30（全球標高データ）

```json
{
  "tilejson": "3.0.0",
  "name": "Copernicus DEM GLO-30",
  "description": "Copernicus Digital Elevation Model at 30m resolution. Global coverage. Vertical datum: EGM2008 geoid height.",
  "attribution": "© ESA Copernicus",
  "tiles": [
    "https://example.org/copernicus-dem-glo30/{z}/{x}/{y}.png"
  ],
  "minzoom": 0,
  "maxzoom": 12,

  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "dataRange": { "min": -500, "max": 9000 }
  }
}
```

---

## 5. JSON Schema

以下は `datapng` オブジェクトの JSON Schema（Draft 2020-12）定義である。ルートレベルの `tileSize` フィールドは TileJSON ルートオブジェクトのプロパティであり、本スキーマの対象外である。

> **注記**: `$id` はドラフト段階のプレースホルダーであり、正式公開時に実際のホスティングURLへ変更する。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.org/tilejson-datapng-extension/0.5.0/schema.json",
  "title": "TileJSON DataPNG Extension",
  "description": "データPNG仕様に基づくグリッドPNGタイルセットメタデータの TileJSON 拡張",
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": {
      "type": "string",
      "enum": ["numerical", "palette"],
      "description": "データPNGの種別"
    },
    "factor": {
      "type": "number",
      "description": "数値PNG用の変換係数 f"
    },
    "offset": {
      "type": "number",
      "description": "数値PNG用のオフセット o"
    },
    "unit": {
      "type": "string",
      "description": "変換後の値の単位"
    },
    "invalidColor": {
      "type": "array",
      "items": { "type": "integer", "minimum": 0, "maximum": 255 },
      "minItems": 3,
      "maxItems": 3,
      "description": "追加無効色 [r, g, b]"
    },
    "dataRange": {
      "type": "object",
      "properties": {
        "min": { "type": "number", "description": "デコード後の最小期待値" },
        "max": { "type": "number", "description": "デコード後の最大期待値" }
      },
      "description": "デコード後の値の期待範囲"
    },
    "precision": {
      "type": "number",
      "exclusiveMinimum": 0,
      "description": "元データの有効な最小単位"
    },
    "legend": {
      "oneOf": [
        {
          "type": "string",
          "format": "uri",
          "description": "外部凡例JSON URL"
        },
        {
          "type": "object",
          "required": ["items"],
          "properties": {
            "title": { "type": "string" },
            "items": {
              "type": "array",
              "items": {
                "type": "object",
                "required": ["r", "g", "b", "title"],
                "properties": {
                  "r":           { "type": "integer", "minimum": 0, "maximum": 255 },
                  "g":           { "type": "integer", "minimum": 0, "maximum": 255 },
                  "b":           { "type": "integer", "minimum": 0, "maximum": 255 },
                  "title":       { "type": "string" },
                  "description": { "type": "string" }
                },
                "additionalProperties": true
              }
            }
          },
          "additionalProperties": true
        }
      ],
      "description": "パレットPNG凡例情報（インラインまたはURL）"
    },
    "support": {
      "type": "object",
      "required": ["type"],
      "properties": {
        "type":   { "type": "string", "enum": ["point", "block"] },
        "anchor": { "type": "string", "enum": ["northwest", "center"] }
      },
      "additionalProperties": false,
      "description": "格納値の support（点/面）。anchor は type が point の場合のみ有効"
    }
  },
  "allOf": [
    {
      "if": {
        "required": ["type"],
        "properties": { "type": { "const": "numerical" } }
      },
      "then": {
        "properties": {
          "factor":   { "default": 1 },
          "offset":   { "default": 0 }
        }
      }
    },
    {
      "if": {
        "required": ["type"],
        "properties": { "type": { "const": "palette" } }
      },
      "then": {
        "required": ["type", "legend"]
      }
    }
  ]
}
```

---

## 6. クライアント実装ガイドライン

### 6.1 パース手順

1. TileJSON をパースし、`datapng` キーの有無を確認する。
2. `datapng` が存在しない場合、通常のラスタータイルとして扱う。
3. `datapng.type` に応じたデコーダを選択する。
4. 数値PNGの場合、§3.2 の無効値判定手順に従い、有効なピクセルに対して `factor`・`offset` による値変換を行う。
5. パレットPNGの場合、`legend`（インラインまたはURLフェッチ）を使って RGB 完全一致による凡例検索を行う。

### 6.2 後方互換性

- `datapng` を認識しないクライアントはこのキーを無視する（TileJSON 3.0.0 §1 に準拠）。
- タイル画像自体は有効な PNG であるため、画像としての表示は引き続き可能。

### 6.3 凡例項目の拡張性

凡例項目オブジェクトの拡張ルールについては §3.3.1 を参照。

---

## 7. フィールド一覧（早見表）

| フィールド | 型 | 数値PNG | パレットPNG | デフォルト |
|-----------|-----|:-------:|:-----------:|-----------|
| `type` | String | ✔ | ✔ | — (必須) |
| `factor` | Number | ○ | — | `1` |
| `offset` | Number | ○ | — | `0` |
| `unit` | String | ○ | — | — |
| `invalidColor` | [r,g,b] | ○ | — | — |
| `dataRange` | Object | ○ | — | — |
| `precision` | Number | ○ | — | — |
| `legend` | Obj/URL | — | ✔ | — |
| `support` | Object | ○ | ○ | — |

✔ = 必須、○ = 任意、— = 該当なし

---

## 8. 今後の検討事項

- **リストPNG（List PNG）の対応**: 固定長レコードデータ（点群PNG（Point Cloud PNG）を含む）への対応。`type` 値の追加とカラム定義スキーマ
- **外部エンコーディング互換**: Mapbox Terrain-RGB 等への `encoding` 値の追加
- **タイルサイズ**: 256px / 512px の明示的記述
- **凡例フォーマットの拡張**: 数値範囲による連続的な色分け凡例への対応

---

## ライセンス

本仕様案は CC BY 4.0 で公開する。本仕様に準拠したソフトウェアの実装・配布にあたって、本仕様の著作権者へのクレジット表示は不要である。
