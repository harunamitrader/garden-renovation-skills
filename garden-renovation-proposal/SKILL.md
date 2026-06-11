---
name: garden-renovation-proposal
description: 造園業者向け。現況写真と要望から、施主に渡せる「お庭リフォーム提案ページ」（モダンデザインの単一HTML、スマホ対応、BEFORE/AFTER比較スライダー付き）を生成する。完成予想図は garden-renovation-visualizer、平面図は garden-renovation-siteplan で画像生成し、提案ページ本体は HTML テンプレートに実画像をそのまま埋め込む。「この写真と要望で提案資料を作って」「○○様邸の提案ページにまとめて」のような指示で使う。金額・工期はユーザーが提示した値のみ記載し、絶対に創作しない。
---

# Garden Renovation Proposal

現況写真と要望から、施主提出用の提案ページ（モダンデザインの単一 HTML）を生成するスキル。

設計方針（ハイブリッド方式）:
- **ビジュアル（完成予想図・平面図）= 画像生成**: `garden-renovation-visualizer` / `garden-renovation-siteplan` の成果物を使う
- **提案ページ本体 = コード組版**: `assets/proposal.template.html` を埋めて完成させる
- 理由: image_gen でページごと生成すると施主の実写真・生成済み完成予想図が再描画されて別物になり、金額・施主名などの事務テキストにも誤字リスクが出るため。実画像の埋め込みとテキストの正確性はコード組版で保証する

## トリガー

- 「この写真と要望で提案資料を作って」「○○様邸の提案ページにまとめて」のような指示
- 入力（いずれか）:
  1. `garden-renovation-visualizer` の出力フォルダ（`before.png` / `after.png` / `plan.md`、あれば `siteplan.png`）
  2. 現況写真 + 自然言語の要望（この場合は先に完成予想図の生成から行う）

## 出力

```
garden-design-output/{YYYY-MM-DD}-{slug}/
├── before.png / after.png / plan.md   # 既存 or このスキル実行中に生成
├── siteplan.png                       # 平面図セクションを含める場合のみ
└── proposal.html                      # 提案ページ本体（画像は同フォルダ相対参照）
```

出力先は、ユーザーが指定した場所があればそこ。無ければ**カレントディレクトリ直下の `garden-design-output/`** フォルダを使う。

`proposal.html` をブラウザで開けばそのまま施主に見せられる。フォルダごと渡せばオフラインでも表示できる。

## ワークフロー

### Step 1: 素材の確認・生成

- 出力フォルダに `after.png` と `plan.md` があればそれを使う
- 無ければ `garden-renovation-visualizer` のワークフローに従って先に生成する（写真が無ければ写真の提供を求めて終了）
- 平面図セクションはオプション。`siteplan.png` が既にあれば含める。無ければ「平面図セクションも入れますか？」と確認し、必要なら `garden-renovation-siteplan` のワークフローで生成する

### Step 2: 会社情報の取得

- このスキル自身のフォルダ内の `assets/company.json`（例: `~/.agents/skills/garden-renovation-proposal/assets/company.json`）があれば読み込んで使う
- 無ければ会社名・担当者名・連絡先（電話 / メール、任意）を質問し、回答を同ファイルに保存してよいか確認する。保存形式:

```json
{ "name": "会社名", "person": "担当者名", "tel": "", "email": "", "note": "" }
```

- 「会社情報なし」でも進められる（発行者欄を省略する）

### Step 3: 提案内容の整理

`plan.md` とユーザーの指示から以下を整理する:

- **施主名**: 指示に無ければ質問する（「未記入でよい」も可。その場合は「お客様邸」とする）
- **現況の課題**: 要望の背景を 3 点前後に箇条書き化（写真と要望から導けるもののみ）
- **提案ポイント**: 変更内容を施主向けの言葉で 3 点に整理。各ポイントは**短いタイトル（15 字以内）+ 説明文（1〜2 文）**の形にする（plan.md の「変更内容」をベースに、メリットを添える）
- **工事概要・概算金額・工期**: **ユーザーが明示した値のみ**記載する。提示が無ければそのセクションごと省略する。金額・数量・工期を推測で書くことは禁止

### Step 4: HTML の組み立て

- `assets/proposal.template.html` を出力フォルダに `proposal.html` としてコピーし、`{{...}}` プレースホルダを置換する
- 提案ポイントは `{{POINT_n_TITLE}}`（タイトル）と `{{POINT_n_BODY}}`（説明文）のペアで埋める
- オプションブロックは `<!-- BEGIN:xxx -->` 〜 `<!-- END:xxx -->` で囲まれている。使わないブロックは**コメントごと削除**する:
  - `SITEPLAN`: 平面図セクション
  - `ESTIMATE`: 工事概要・概算セクション
  - `ISSUER`: 発行者（会社情報）欄。**ヒーローとフッターの 2 か所**にあるので両方削除する
- 画像参照は同フォルダ相対パス（`before.png` 等）のまま使う
- 課題カード・提案ポイントの個数を変える場合は、既存のマークアップ（`.issue` / `.point`）を複製・削除して調整する
- BEFORE/AFTER スライダーの免責文と、フッターの AI 免責文は**削除しない**

### Step 5: 表示確認

- 利用可能なブラウザツールがあれば `proposal.html` を開いてスクリーンショットで確認する。無ければ Edge ヘッドレスで確認する:

```
# Windows（Edge を探して実行）
$edge = @(
  "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe",
  "C:\Program Files\Microsoft\Edge\Application\msedge.exe"
) | Where-Object { Test-Path $_ } | Select-Object -First 1
& $edge --headless --disable-gpu --window-size=1280,4000 --screenshot="{出力フォルダ}\_check.png" "{出力フォルダ}\proposal.html"

# macOS / Linux
# google-chrome --headless --disable-gpu --window-size=1280,4000 --screenshot=...
```

確認項目:

- [ ] 施主名・会社名・日付・金額（あれば）に誤りがないか
- [ ] ヒーロー・BEFORE/AFTER・平面図に正しい画像が表示されているか（画像切れ・歪みがないか）
- [ ] 省略したオプションセクションの残骸（空セクション・プレースホルダ）が無いか
- [ ] AI 免責文がスライダー下とフッターに残っているか

確認後、`_check.png` は削除する。

### Step 6: 報告

- 保存先フルパス一覧とページの構成（含めたセクション）を報告する
- 「`proposal.html` をブラウザで開いてください。デザインを変えたい場合は `proposal.html` を直接編集できます」と案内する

## 能力の限界・運用上の注意（ユーザーへの説明事項）

- 完成予想図・平面図は AI 生成のため、施工後の実際の仕上がり・寸法とは異なる。ページにはその旨の免責を必ず入れてあり、削除しない
- 金額・工期はこのスキルでは一切算出しない。見積りは別途、業務上の積算で行う前提
- BEFORE/AFTER スライダーは画像 2 枚の構図が近いほど効果的。構図が大きくずれている場合はその旨を報告する

## エラー時の動作

| 状況 | 動作 |
|------|------|
| 写真も既存フォルダも無い | 現況写真の提供を求めて終了 |
| 完成予想図の生成に失敗 | garden-renovation-visualizer のエラー時の動作に従う |
| 金額・工期を聞かれていないのに資料に必要と思われる | 推測で書かず、「概算セクションは金額の提示があれば追加できます」と案内して省略する |

## 注意

- 既存の `proposal.html` がある場合は上書きせず `proposal-v2.html` のように版を分ける
- `assets/proposal.template.html` は編集しない。カスタマイズは出力フォルダ側の `proposal.html` で行う
- 施主の個人情報（名前・住所）を出力フォルダ以外へコピーしない
