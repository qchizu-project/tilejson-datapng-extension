# TileJSON DataPNG Extension

データPNG（数値PNG・パレットPNG）をタイルとして配信する際に必要なメタデータを、[TileJSON 3.0.0](https://github.com/mapbox/tilejson-spec/tree/master/3.0.0) に記述するための拡張仕様（案）です。

**ステータス:** Draft v0.7.0 ／ **ライセンス:** [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.ja)

## 仕様書

- 📄 [**tilejson-datapng-extension.ja.md**](./tilejson-datapng-extension.ja.md)（日本語版）
- 🧩 [**schema/datapng-0.7.0.schema.json**](./schema/datapng-0.7.0.schema.json)（`datapng` オブジェクトの JSON Schema）

英語版（`tilejson-datapng-extension.en.md`）は今後追加予定です。

## 概要

TileJSON 3.0.0 は地図タイルセットの汎用メタデータ規格ですが、**データPNG**（[データPNG](https://gsj-seamless.jp/labs/datapng/) 仕様に基づく数値PNG・パレットPNG）をタイル配信する際に必要な属性情報を記述する手段を持ちません。

本拡張は、TileJSON が未知キーを無視する性質を利用し、ルートオブジェクトに **`datapng`** キーを追加します。これにより、後方互換性を保ったまま、クライアントがタイルのデコード・描画に必要な情報（種別・変換係数・単位・無効値・凡例など）を事前に取得できます。

「データPNG」という名称ですが、格納形式は PNG のほか **WebP（可逆圧縮）**も使用できます。

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

各フィールドの定義・変換式・JSON Schema は[仕様書](./tilejson-datapng-extension.ja.md)を参照してください。

## 参照

| 仕様 | URL |
|------|-----|
| TileJSON 3.0.0 | https://github.com/mapbox/tilejson-spec/tree/master/3.0.0 |
| データPNG | https://gsj-seamless.jp/labs/datapng/ |
| グリッドPNGタイル仕様 (v0.1) | https://gsj-seamless.jp/labs/datapng/gridpngtileSpec.html |

## ライセンス

本仕様案は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/deed.ja)（パブリックドメイン献呈）で公開します。利用・実装・配布にあたり、著作権者へのクレジット表示は不要です。
