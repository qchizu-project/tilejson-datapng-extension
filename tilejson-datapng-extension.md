# TileJSON DataPNG Extension (Draft)

**バージョン: 0.2.0 (2026-03-20)**

産総研データPNG仕様に基づくタイルセットのメタデータを TileJSON 3.0.0 に記述するための拡張仕様（案）。

---

## 1. 概要

TileJSON 3.0.0 は地図タイルセットの汎用メタデータ規格だが、産総研が策定した **データPNG**（数値PNG・パレットPNG）をタイルとして配信する場合に必要な属性情報を記述する手段を持たない。本拡張は、TileJSON の仕様（未知キーを無視する）を利用し、`datapng` キーを追加することでクライアントがタイルのデコード・描画に必要な情報を事前に取得できるようにする。

### 1.1 設計方針

- TileJSON 3.0.0 との**後方互換性**を維持する（未知キーとして無視可能）。
- 産総研データPNG仕様（グリッドPNGタイル仕様 v0.1）に準拠する。
- 本バージョンでは**グリッドPNG**（数値PNG・パレットPNG）を対象とする。
- 測量年次・データソース等の自由記載は TileJSON 既存の `description` フィールドを使用する。

### 1.2 対象外（将来拡張）

以下は本バージョンの対象外とし、将来バージョンで検討する。

- 点群PNG（Point Cloud PNG）
- ベクトル型グリッドPNG（24ビット分割による多チャネル表現）
- Mapbox Terrain-RGB 等の外部エンコーディング互換

### 1.3 参照仕様

| 仕様 | URL |
|------|-----|
| TileJSON 3.0.0 | https://github.com/mapbox/tilejson-spec/tree/master/3.0.0 |
| データPNG | https://gsj-seamless.jp/labs/datapng/ |
| グリッドPNGタイル仕様 | https://gsj-seamless.jp/labs/datapng/gridpngtileSpec.html |
| 全国の標高成果の改定 | https://www.gsi.go.jp/sokuchikijun/hyoko2024rev.html |
| JGD2024の座標系コード | https://www.wingfield.gr.jp/archives/5692 |

---

## 2. 拡張キー `datapng`

TileJSON のルートオブジェクトに **`datapng`** キー（Object）を追加する。

```jsonc
{
  "tilejson": "3.0.0",
  "tiles": [ "https://tiles.gsj.jp/tiles/elev/mixed/{z}/{y}/{x}.png" ],
  "minzoom": 0,
  "maxzoom": 15,

  "datapng": {
    // ── 本拡張で定義するフィールド群 ──
  }
}
```

---

## 3. フィールド定義

### 3.1 `datapng.type` — データPNG種別

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `type` | String (enum) | **REQUIRED** | データPNGの種類 |

指定可能な値:

| 値 | 意味 |
|----|------|
| `"numerical"` | 数値PNG（連続値スカラー型） |
| `"palette"` | パレットPNG（離散色凡例型） |

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
| `invalidColor` | Array[3] of int | OPTIONAL | — | 追加無効色 `[r, g, b]`。透明ピクセルに加えて無効値として扱う色 |

変換式:

```
r' = (r < 128) ? r : r - 256
rawValue = r' × 65536 + g × 256 + b
v = factor × rawValue + offset
```

> **補足**: 完全に透明なピクセル（不透明度 0）は常に無効値として扱う。`invalidColor` は透明ピクセルとは別に、不透明だが無効とみなすべき色を指定するもの（例: 国土地理院標高タイルの `[128, 0, 0]`）。

**例: 国土地理院標高タイル（rawValue をメートル単位に変換）**

```json
{
  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "offset": 0,
    "unit": "m",
    "invalidColor": [128, 0, 0]
  }
}
```

### 3.3 パレットPNG用フィールド

`type` が `"palette"` の場合に使用するフィールド群。

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `legend` | Object or String | OPTIONAL | 凡例情報（インラインまたはURL参照） |

#### 3.3.1 凡例のインライン定義

産総研 JSON凡例フォーマットに準拠した構造をそのまま埋め込む。

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

> **仕様上の拡張性**: 産総研JSON凡例フォーマットに従い、凡例項目オブジェクトに上記以外の任意のメンバーを追加することができる。クライアントは処理できないメンバーを無視しなければならない（MUST）。これにより、シンボル画像URL、数値範囲、表示順序等をアプリケーション固有に追加できる。

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

凡例データが大きい場合はURLで参照する。`legend` の値が文字列の場合、クライアントはそのURLから JSON凡例フォーマットを取得する。

```json
{
  "datapng": {
    "type": "palette",
    "legend": "https://gbank.gsj.jp/seamless/v2/api/1.2/legend.json"
  }
}
```

### 3.4 `datapng.verticalCrs` — 鉛直座標参照系

標高タイル等、値が鉛直方向の物理量を表す場合に、その基準面を示す。

| キー | 型 | 必須 | デフォルト | 説明 |
|------|----|------|-----------|------|
| `verticalCrs` | String | OPTIONAL | — | 鉛直座標参照系（EPSG コード形式を推奨） |

値には EPSG コード形式（`"EPSG:NNNNN"`）を推奨する。各国・地域の鉛直基準面や全球ジオイドモデルに対応する EPSG コードを指定できる。

主な指定例:

| 値 | 意味 |
|----|------|
| `"EPSG:3855"` | EGM2008 ジオイド高（全球） |
| `"EPSG:5773"` | EGM96 ジオイド高（全球） |
| `"EPSG:5703"` | NAVD88 height（北米） |
| `"EPSG:6695"` | JGD2011 vertical height（日本） |
| `"EPSG:105604"` | JGD2024 vertical height（日本） |
| `"EPSG:5782"` | Alicante height（スペイン） |

> **注記**: 鉛直基準面は国・地域ごとに異なり、同一地点でも基準面の違いにより標高値が異なる。タイルデータが依拠する鉛直基準面を `verticalCrs` で明示することで、異なるデータセット間の整合性を確保できる。例えば、日本では2025年4月にJGD2011からJGD2024へジオイド・モデルが移行され、同一地点で最大数十cm程度の差が生じうる（参照: [全国の標高成果の改定](https://www.gsi.go.jp/sokuchikijun/hyoko2024rev.html)、[JGD2024の座標系コード](https://www.wingfield.gr.jp/archives/5692)）。なお、`EPSG:105604` は2025年12月時点でEPSGへの正式登録が未完了であり、コードが変更される可能性がある。

```json
{
  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "verticalCrs": "EPSG:6695"
  }
}
```

### 3.5 `datapng.pixelMapping` / `datapng.resampling` — ピクセル解釈とリサンプリング

ピクセル値が地理空間上のどの位置・範囲を表すか、またタイル生成時にどのリサンプリングが使用された（または推奨される）かを示す。両フィールドは密接に関連しており、`pixelMapping` がピクセルの空間的意味を定義し、`resampling` がズームレベル間の値の導出方法を示す。

| キー | 型 | 必須 | デフォルト | 説明 |
|------|----|------|-----------|------|
| `pixelMapping` | String (enum) | OPTIONAL | `"area"` | ピクセルの値が地理空間上のどの位置・範囲を表すか |
| `resampling` | String (enum) | OPTIONAL | — | リサンプリングアルゴリズム |

**`pixelMapping` の指定可能な値:**

| 値 | 説明 | 典型的な用途 |
|----|------|-------------|
| `"northwest"` | ピクセル北西端（左上）の点の値を表す | シームレス標高タイル |
| `"center"` | ピクセル中央点の値を表す | 一般的な格子データ |
| `"area"` | ピクセル範囲全体の代表値（面積型） | 国土地理院標高タイル、地質図 |

**`resampling` の指定可能な値:**

| 値 | 説明 | 適用例 |
|----|------|--------|
| `"nearest"` | 最近隣法（Nearest Neighbor） | 一般的な数値PNG |
| `"northwest"` | 左上法（4ピクセルのうち北西を採用） | シームレス標高タイル |
| `"average"` | 平均値法 | 国土地理院標高タイル |
| `"majority"` | 多数決法（面積最大のカテゴリを採用） | シームレス地質図 |
| `"bilinear"` | 双線形補間 | 連続値の平滑化 |

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
    "verticalCrs": "EPSG:6695",
    "pixelMapping": "northwest",
    "resampling": "northwest"
  }
}
```

### 4.2 国土地理院 標高タイル PNG形式

```json
{
  "tilejson": "3.0.0",
  "name": "地理院標高タイル（DEM5A）",
  "description": "国土地理院 基盤地図情報数値標高モデル 5mメッシュ。標高基準面: 東京湾平均海面。",
  "attribution": "国土地理院",
  "tiles": [
    "https://cyberjapandata.gsi.go.jp/xyz/dem5a_png/{z}/{x}/{y}.png"
  ],
  "minzoom": 1,
  "maxzoom": 15,

  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "invalidColor": [128, 0, 0],
    "verticalCrs": "EPSG:6695",
    "pixelMapping": "area",
    "resampling": "average"
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
    },
    "resampling": "nearest"
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
    "legend": "https://gbank.gsj.jp/seamless/v2/api/1.2/legend.json",
    "resampling": "majority"
  }
}
```

---

## 5. JSON Schema

以下は `datapng` オブジェクトの JSON Schema（Draft 2020-12）定義である。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.org/tilejson-datapng-extension/0.2.0/schema.json",
  "title": "TileJSON DataPNG Extension",
  "description": "産総研データPNG仕様に基づくグリッドPNGタイルセットメタデータの TileJSON 拡張",
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
    "verticalCrs": {
      "type": "string",
      "description": "鉛直座標参照系（EPSG コード形式を推奨）"
    },
    "pixelMapping": {
      "type": "string",
      "enum": ["northwest", "center", "area"],
      "default": "area",
      "description": "ピクセル値と地理座標の対応方法"
    },
    "resampling": {
      "type": "string",
      "enum": ["nearest", "northwest", "average", "majority", "bilinear"],
      "description": "タイル作成時のリサンプリング手法"
    }
  },
  "allOf": [
    {
      "if": { "properties": { "type": { "const": "numerical" } } },
      "then": {
        "properties": {
          "factor":   { "default": 1 },
          "offset":   { "default": 0 }
        }
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
4. 数値PNGの場合、`factor`・`offset`・`invalidColor` を用いて値変換を行う。
5. パレットPNGの場合、`legend`（インラインまたはURLフェッチ）を使って凡例検索を行う。

### 6.2 後方互換性

- `datapng` を認識しないクライアントはこのキーを無視する（TileJSON 3.0.0 §1 に準拠）。
- タイル画像自体は有効な PNG/WebP であるため、画像としての表示は引き続き可能。

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
| `legend` | Obj/URL | — | ○ | — |
| `verticalCrs` | String | ○ | — | — |
| `pixelMapping` | String | ○ | ○ | `"area"` |
| `resampling` | String | ○ | ○ | — |

✔ = 必須、○ = 任意、— = 該当なし

---

## 8. 今後の検討事項

- **点群PNG（Point Cloud PNG）** の対応: `type: "pointcloud"` 追加およびカラム定義スキーマ
- **ベクトル型グリッドPNG**: 24ビットを分割して複数チャネルを持つ場合のスキーマ定義
- **外部エンコーディング互換**: Mapbox Terrain-RGB 等への `encoding` 値の追加
- **タイルサイズ**: 256px / 512px の明示的記述
- **凡例フォーマットの拡張**: 数値範囲による連続的な色分け凡例への対応
- **JGD2024 対応**: `EPSG:105604` の EPSG 正式登録を確認次第、コード確定を反映

---

## ライセンス

本仕様案は CC BY 4.0 で公開する。
