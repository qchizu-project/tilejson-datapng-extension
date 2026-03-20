# レビュー指摘事項

## A. 技術的正確性

### A-1. EPSG:6695 の鉛直CRSとしての正確性 [CRITICAL]

`EPSG:6695` は平面直角座標系（Japan Plane Rectangular CS XIX）である可能性があり、鉛直CRSではないかもしれない。JGD2011の鉛直CRSとしてEPSG Registryでの再確認が必要。

### A-2. タイルURL座標順序の混在 [CRITICAL]

4.1/4.4 は `{z}/{y}/{x}`、4.2/4.3 は `{z}/{x}/{y}`。4.1は `scheme: "xyz"` を明示しながら `{z}/{y}/{x}` のため、TileJSON仕様上は矛盾して見える。実際のサーバーURLがその順序なら正しいが、読者向けに注記が必要。

### A-3. JSON Schema の `if` 条件に `required` が欠落 [CRITICAL]

```json
"if": { "properties": { "type": { "const": "numerical" } } }
```

`type` が省略されたオブジェクトで `if` が意図せず `true` と評価される。以下のように `required` を追加すべき:

```json
"if": { "required": ["type"], "properties": { "type": { "const": "numerical" } } }
```

### A-4. `invalidColor` の評価タイミングが未定義 [CRITICAL]

`[128, 0, 0]` のピクセルに対して、変換式を適用する前にチェックするのか後にチェックするのかが明記されていない。実装ガイドラインに「invalidColor チェックは値変換より先に行う」と記載すべき。

### A-5. `rawValue` の数値範囲が未記載 [WARNING]

変換式はあるが、rawValueの範囲（-8,388,608 〜 8,388,607、つまり24ビット符号付き整数の全域）が明記されていない。

### A-6. `EPSG:105604` はESRIコード体系の可能性 [WARNING]

`105xxx` 台はESRI独自のコード体系であり、EPSGの公式コードとは異なる可能性がある。名前空間を明示するか、正式登録前の暫定的な扱いをより明確にすべき。

### A-7. `wingfield.gr.jp` は個人サイト [WARNING]

仕様の参照先として個人ブログは信頼性・永続性に課題がある。公的機関の資料やEPSG Registryへのリンクが望ましい。

---

## B. 仕様の完全性

### B-1. アルファチャンネルの扱いが未定義 [CRITICAL]

「完全に透明なピクセル（不透明度 0）は常に無効値」とあるが、半透明ピクセル（0 < A < 255）の扱いが未定義。A=0 のみ無効なのか、閾値があるのかを明記すべき。

### B-2. `invalidColor` マッチング時のアルファ値の考慮 [CRITICAL]

ピクセルが `[128, 0, 0, 200]`（半透明）の場合、`invalidColor: [128, 0, 0]` にマッチするかが不明確。RGBの完全一致のみで判定するのか、アルファ値の条件はあるのかを明記すべき。

### B-3. `invalidColor` が1色のみ [CRITICAL]

`Array[3]` の固定長配列のため、複数の無効色を指定できない。意図的な制約であれば明記が必要。将来拡張として `Array of Array[3]` への変更を検討するか、今後の検討事項に記載すべき。

### B-4. 外部凡例URLのフェッチ失敗時の挙動が未規定 [CRITICAL]

HTTPエラー・タイムアウト・CORSエラー等でフェッチが失敗した場合にクライアントが取るべき動作が記述されていない。

### B-5. 型ごとの禁止フィールドが未規定 [CRITICAL]

`numerical` に `legend` を指定した場合、`palette` に `factor`/`offset`/`verticalCrs` を指定した場合の挙動が文章にもSchemaにも定義されていない。早見表の「—（該当なし）」との矛盾。

### B-6. `resampling` の用途が曖昧 [WARNING]

「タイル生成時に使用された」情報なのか「クライアントに推奨する」指示なのかが不明確。セマンティクスを明確にすべき。

### B-7. `legend` 省略時のパレットPNG描画方法が未規定 [WARNING]

`type: "palette"` で `legend` が OPTIONAL だが、凡例がない場合の描画方法（PNGのRGBをそのまま色として表示？）が記述されていない。

### B-8. RFC 2119 キーワードの不統一 [WARNING]

MUST/SHOULD/MAY の使用が散発的で、仕様書冒頭にRFC 2119への参照宣言がない。

### B-9. バージョニング戦略が未記述 [WARNING]

0.x → 1.0 への移行条件、`datapng` オブジェクト内にバージョン番号を持たない設計の意図が未記載。

### B-10. `pixelMapping` のデフォルト `"area"` の妥当性 [WARNING]

全ての numerical データにデフォルト `"area"` が適切とは限らない。省略時に誤った解釈が生じるリスクがある。

---

## C. 国際仕様としての設計

### C-1. 仕様書が日本語のみ [CRITICAL]

国際仕様には英語版が必須。JSON Schema の description も日本語になっている。

### C-2. `datapng` キー名が産総研固有用語 [CRITICAL]

国際的な文脈では "datapng" の意味が伝わらない。`encodedRaster` 等の汎用名を検討すべきか。

### C-3. 凡例フォーマットが産総研固有 [CRITICAL]

OGC SLD / Mapbox Style 等の国際標準との互換性への言及がない。`r`/`g`/`b` 分離形式は国際的には `#RRGGBB` が慣行的。

### C-4. `type` の値名が産総研用語 [WARNING]

`"numerical"` / `"palette"` は産総研の用語体系。国際的には `"continuous"` / `"categorical"` が一般的。

### C-5. 例が全て日本のデータセット [WARNING]

SRTM / Copernicus DEM 等の全球データ例があると国際利用者に親切。

### C-6. ライセンス CC BY 4.0 の適切性 [WARNING]

実装者にクレジット義務を課す解釈が可能。TileJSON本体は CC0。仕様準拠時の帰属義務の有無を明記すべき。

### C-7. JSON Schema の `$id` が `example.org` のまま [WARNING]

正式な公開先URLの計画、またはドラフト段階である旨の注記が必要。
