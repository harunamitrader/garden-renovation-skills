---
name: garden-renovation-siteplan
description: garden-renovation-visualizer が出力した改修プラン（plan.md と完成予想図）、または庭写真と要望から、外構の平面図（真上から見た 2D サイトプラン）を画像生成する。gpt-image-2 の高精度な日本語テキスト描画を活かし、凡例・方位・ラベル付きの図面風イラストを作る。「このプランの平面図を作って」のような指示で使う。CAD 図面の代替ではない。
---

# Garden Renovation Siteplan

改修プランをもとに、真上から見た外構平面図（サイトプラン）を画像生成するスキル。
Codex の built-in `image_gen` ツール（gpt-image-2）の **generate モード**（参照画像つき）を使う。

## トリガー

- 「このプランの平面図を作って」「garden-renovation-siteplan で○○の平面図を」のような指示
- 入力（優先順）:
  1. `garden-renovation-visualizer` の出力フォルダ（`plan.md` + `after.png` + `before.png`）
  2. 完成予想図 or 庭写真 + 自然言語の説明

## 出力

```
{入力フォルダ}\siteplan.png      # 入力が visualizer の出力フォルダの場合
```

入力フォルダが無い場合は、カレントディレクトリ直下の `garden-design-output/{YYYY-MM-DD}-{slug}/siteplan.png` として新規作成する。

## ワークフロー

### Step 1: 入力の収集

- `plan.md` があれば読み、「変更内容」「保持した要素」「敷地の位置関係」「実寸の情報」を取得する
- 画像（`after.png`、あれば `before.png` も）を built-in `view_image` で読み込み、配置の参照にする
- `plan.md` も画像も無い場合は、敷地の構成（建物の位置、庭の要素、おおよその位置関係）を質問してから進む

### Step 2: レイアウトの言語化

平面図に描く要素を確定し、**真上から見た配置として**言語化する:

- 建物の輪郭（フットプリント）と玄関位置
- 改修後の要素（デッキ・園路・植栽帯・駐車場など）の位置・形・隣接関係
- 境界（塀・フェンス・道路側）
- 写真は斜めから撮られているため、奥行きの距離感は推定になる。**位置関係（トポロジー）を優先し、寸法は確定情報がある場合のみ採用する**

不明な位置関係（例: 写真に写っていない建物の裏側）は描かないか、ユーザーに確認する。

### Step 3: 生成プロンプトの構築

英語で、以下の構造のプロンプトを組み立てる（ラベル文字列は日本語のまま埋め込む）:

```
Use case: 2D landscape architecture site plan (top-down view)
Asset type: flat architectural drawing, NOT photorealistic, NOT 3D, NOT perspective
Primary request: top-down site plan of a renovated residential garden with this layout: [Step 2 のレイアウト記述]
Include:
- building footprint drawn with hatching, labeled 「建物」
- [各要素 + 日本語ラベル。例: wooden deck labeled 「ウッドデッキ」]
- legend box (凡例) at the bottom-left listing all symbols with Japanese labels
- north arrow (方位記号 N)
- [実寸があれば] dimension lines with measurements, [無ければ] no dimension numbers
Style: clean thin line drawing with soft pastel fills, white background, Japanese residential exterior plan (外構図) conventions, top-down orthographic view only
Text: Japanese labels, high readability, consistent font size
Disclaimer in small gray text at the bottom-right: ※AIが生成した概念図です。寸法・縮尺は目安であり正確ではありません。
Avoid: perspective, shadows implying 3D, photorealistic textures, English labels, invented rooms inside the building
```

- 出力サイズは敷地の縦横比に合わせて `1536x1024` / `1024x1536` / `1024x1024` から選ぶ
- 参照画像として `after.png`（あれば `before.png` も）を添えると配置の整合が上がる。プロンプト内で各画像の役割を明示する（例: "image 1 is the renovated garden photo, use it only as a layout reference"）

### Step 4: 検証

- [ ] 真上視点になっているか（パース・影が混ざっていないか）
- [ ] 要素の位置関係が `plan.md` / 写真と矛盾していないか
- [ ] 日本語ラベル・凡例の文字が正確か（gpt-image-2 は高精度だが必ず目視確認する）
- [ ] 免責注記が入っているか
- [ ] 頼んでいない要素（架空の部屋・設備）が描き込まれていないか

崩れがあれば 1 回につき修正点を 1 つに絞ってイテレーションする。文字の誤りは「fix only the text 「…」, keep everything else identical」のように指示する。

### Step 5: 保存と報告

- `siteplan.png` を出力フォルダに保存する（`~/.codex/generated_images/` に置きっぱなしにしない）
- `plan.md` が存在する場合、末尾に `## 平面図` セクションを追記して `siteplan.png` への参照と生成日を記録する
- 保存先フルパス・使用プロンプト・検証結果を報告する

## 能力の限界（ユーザーへの説明事項）

初回出力時に以下を一言添える:

- これは**概念図**であり、CAD 図面・施工図の代替にはならない。寸法・縮尺・面積はあてにできない
- 斜め写真からの真上配置は推定。要素の位置関係は再現できるが、距離・比率は不正確になりうる
- 凡例やラベルの文字は高精度だが、細かい寸法数字の羅列は誤りが出やすいので最小限にしている
- 正確な図面が必要なら、この概念図を下敷きに専門業者・CAD で清書する想定

## エラー時の動作

| 状況 | 動作 |
|------|------|
| plan.md も画像も説明も無い | 敷地構成を質問して、回答が無ければ終了 |
| 2 回イテレーションしても真上視点にならない | 現状の最良版を提示して限界を説明する |
| 日本語ラベルの文字化け・誤字が直らない | ラベル数を減らして再生成、それでも駄目なら誤字箇所を報告する |
| image_gen 失敗 | エラー内容を報告。CLI fallback は提案のみ（勝手に実行しない） |

## 注意

- 既存の `siteplan.png` がある場合は上書きせず `siteplan-v2.png` のように版を分ける
- 建物内部の間取りは描かない（外構平面図の範囲外）
- 透明背景は使わない（gpt-image-2 非対応のため。白背景固定）
