# TileJSON DataPNG Extension

データPNG（数値PNG・パレットPNG）をタイルとして配信する際に必要なメタデータを、[TileJSON 3.0.0](https://github.com/mapbox/tilejson-spec/tree/master/3.0.0) に記述するための拡張仕様（案）です。

**ステータス:** Draft v0.7.0 ／ **ライセンス:** [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.ja)

## 仕様書

- 📄 [**tilejson-datapng-extension.ja.md**](./tilejson-datapng-extension.ja.md)（日本語版）
- 🧩 [**schema/datapng-0.7.0.schema.json**](./schema/datapng-0.7.0.schema.json)（`datapng` オブジェクトの JSON Schema）

英語版（`tilejson-datapng-extension.en.md`）は今後追加予定です。

## 概要

TileJSON 3.0.0 は地図タイルセットの汎用メタデータ規格ですが、**データPNG**（[データPNG](https://gsj-seamless.jp/labs/datapng/) 仕様に基づく数値PNG・パレットPNG）をタイル配信する際に必要な属性情報を記述する手段を持ちません。

本拡張は、TileJSON が未知キーを無視する性質を利用し、ルートオブジェクトに **`datapng`** キーを追加します。これにより、後方互換性を保ったまま、クライアントがタイルのデコード・描画に必要な情報（種別・変換係数・単位・無効値・凡例など）を事前に取得できます。あわせて、タイル画像のピクセルサイズ（`tileSize`）もルートに追加します。

タイル画像の**標準形式は WebP（可逆圧縮）**で、PNG も許容します。形式はタイル URL の拡張子が示すため、専用のフィールドは設けていません。「データPNG」という名称は先行仕様との連続性のために維持していますが、格納形式は PNG に限りません。

## 最小例

```json
{
  "tilejson": "3.0.0",
  "tiles": ["https://tiles.gsj.jp/tiles/elev/mixed/{z}/{y}/{x}.png"],
  "minzoom": 0,
  "maxzoom": 15,
  "tileSize": 256,
  "datapng": {
    "type": "numerical",
    "factor": 0.01,
    "unit": "m",
    "invalidColor": [128, 0, 0],
    "dataRange": { "min": -500, "max": 9000 }
  }
}
```

（アルファチャンネルを持たない PNG タイルの例。アルファチャンネルを持つタイルでは無効値をアルファ 0 で表し、`invalidColor` は指定しません。）

各フィールドの定義・変換式・JSON Schema は[仕様書](./tilejson-datapng-extension.ja.md)を参照してください。

## 参照

| 仕様 | URL |
|------|-----|
| TileJSON 3.0.0 | https://github.com/mapbox/tilejson-spec/tree/master/3.0.0 |
| データPNG | https://gsj-seamless.jp/labs/datapng/ |
| グリッドPNGタイル仕様 (v0.1) | https://gsj-seamless.jp/labs/datapng/gridpngtileSpec.html |

## 実装

| 実装 | 説明 |
|------|------|
| [datapng-tiler](https://github.com/qchizu-project/datapng-tiler) | 本仕様に準拠したタイルと TileJSON を生成する CLI / Python ライブラリ（MIT） |

## ライセンス

本仕様案は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.ja)（パブリックドメイン献呈）で公開します。利用・実装・配布にあたり、著作権者へのクレジット表示は不要です。
