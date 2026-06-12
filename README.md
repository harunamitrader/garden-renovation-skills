<div align="center">
  <img src="docs/header.png" alt="garden-renovation-skills" width="640">
</div>

# garden-renovation-skills

> **Codex Agent Skills** — 現況写真と自然言語の要望から、庭・外構のリフォーム提案をAIで生成するスキル集。

造園・外構業者向けの Codex スキルを 3 本セットで提供します。**`garden-renovation-proposal` に話しかけるだけで、完成予想図・平面図の生成から提案ページの出力まで一気通貫で完結します。**

---

## デモ

**← BEFORE / AFTER →**

![BEFORE/AFTER比較](docs/sample/before-after.png)

| 現況写真 | 完成予想図 |
|---|---|
| 土の駐車スペース・殺風景な外構 | 2台用カーポート＋タイル舗装のモダン外構 |

**平面図（AI生成）**

![平面図](docs/sample/siteplan.png)

**→ [提案ページのサンプルを見る](https://harunamitrader.github.io/harunami_AI_base/showcase/garden-carport-proposal/)**

---

## 使い方

### 基本的な流れ（これだけでOK）

**`garden-renovation-proposal` スキルに写真と要望を渡すだけ。** 完成予想図・平面図の生成は内部で自動的に行われます。

```
この庭の写真で提案ページを作って。
要望：2台用カーポートを設置して、タイル舗装にしたい。
```

または明示的にスキルを指定する場合：

```
$garden-renovation-proposal
（写真を添付）
2台用カーポートと植栽でモダンな外構にしたい。山田様邸。
```

**内部で自動実行される処理：**

```
garden-renovation-proposal（エントリーポイント）
  ├─ garden-renovation-visualizer  → 完成予想図を生成
  ├─ garden-renovation-siteplan    → 平面図を生成（任意）
  └─ proposal.template.html に埋め込み → proposal.html を出力
```

**出力されるファイル：**

```
garden-design-output/{YYYY-MM-DD}-{案件名}/
├── before.png      # 元写真
├── after.png       # 完成予想図
├── siteplan.png    # 平面図（生成した場合）
├── plan.md         # 改修プランの記録
└── proposal.html   # 施主に渡す提案ページ ← これが最終成果物
```

`proposal.html` はブラウザで開くだけ。フォルダごと渡せばオフラインでも閲覧できます。

### 各スキル単体での利用

提案ページは不要で完成予想図だけ欲しい、平面図だけ作り直したい、といった場合は各スキルを単独で呼び出せます。

| スキル | 単独での呼び出し例 |
|---|---|
| `garden-renovation-visualizer` | `この庭の写真から完成予想図だけ作って` |
| `garden-renovation-siteplan` | `このプランの平面図だけ作って` |
| `garden-renovation-proposal` | `この before/after から提案ページを作って（写真生成済みの場合）` |

---

## スキル構成

```
garden-renovation-skills/
├── garden-renovation-proposal/     # エントリーポイント（ここから始める）
│   ├── SKILL.md
│   └── assets/
│       └── proposal.template.html
├── garden-renovation-visualizer/   # 完成予想図の生成（proposal から呼ばれる）
│   └── SKILL.md
└── garden-renovation-siteplan/     # 外構平面図の生成（proposal から呼ばれる）
    └── SKILL.md
```

---

## スキル詳細

### garden-renovation-proposal（エントリーポイント）

**写真と要望だけ渡せば提案ページまで仕上がる**、メインスキルです。内部で visualizer・siteplan を呼び出し、生成した画像をそのまま HTML テンプレートに埋め込みます。

**特徴**
- モダンデザイン、スマホ対応、スクロール 1 ページ完結
- **BEFORE/AFTER 比較スライダー**付き（ドラッグで写真を比較）
- 完成予想図・平面図は `img` タグで埋め込むだけなので再描画なし（AI 誤りが混入しない）
- 会社情報は `assets/company.json` に保存して使い回し可能
- 金額・工期はユーザーが提示した値のみ記載（推測で書かない）

---

### garden-renovation-visualizer（完成予想図）

現況写真と自然言語の要望から **改修後の完成予想図** を生成します。proposal から呼ばれますが、単独でも使えます。

**特徴**
- Codex の built-in `image_gen`（gpt-image-2）の **edit モード** を使用
- 建物・フェンス・隣地構造物など変更対象外の部分を写真のまま保持
- 変更したい部分だけをプロンプトで指定して置き換え
- 生成後に建物形状・窓の数・パースの保持を検証し、崩れがあれば修正イテレーション

**能力の限界**
- 編集は再生成ベースのため、建物の形状・構図レベルでは高精度に保持されますが、壁のテクスチャや葉の細部はピクセル単位では一致しません
- 出力解像度の実用上限は約 2K
- 一度に変更する項目が 4 つを超えると保持精度が下がるため、スキル内で 2 段階に分割します

---

### garden-renovation-siteplan（平面図）

`plan.md` と完成予想図をもとに **外構平面図（真上視点の 2D サイトプラン）** を生成します。proposal から呼ばれますが、単独でも使えます。

**特徴**
- gpt-image-2 の高精度な日本語テキスト描画（99%超）を活かし、ラベル・凡例・方位記号を画像内に直接描画
- 斜め写真からの位置関係を再現（寸法・縮尺は概算）
- CAD 図面の代替ではなく、施主への説明用の概念図

---

## セットアップ

### 必要環境
- [Codex CLI](https://developers.openai.com/codex/cli) または Codex デスクトップアプリ
- Codex の image_gen 機能が使えるプラン（CLI / App 共通）

### インストール

**グローバルスキルとして使う場合（全プロジェクトで利用可）**

```bash
cp -r garden-renovation-visualizer ~/.agents/skills/
cp -r garden-renovation-siteplan ~/.agents/skills/
cp -r garden-renovation-proposal ~/.agents/skills/
```

**プロジェクトローカルスキルとして使う場合（チームで共有する場合）**

```bash
cp -r garden-renovation-visualizer your-project/.agents/skills/
cp -r garden-renovation-siteplan your-project/.agents/skills/
cp -r garden-renovation-proposal your-project/.agents/skills/
```

### 会社情報の登録（任意）

`~/.agents/skills/garden-renovation-proposal/assets/company.json` を作成しておくと、提案ページに自動で会社名・担当者名が入ります。

```json
{
  "name": "○○造園",
  "person": "担当者名",
  "tel": "000-0000-0000",
  "email": "",
  "note": ""
}
```

---

## CLI / デスクトップアプリの対応

| | Codex CLI | Codex デスクトップアプリ |
|---|---|---|
| スキルの読み込み | ✅ `~/.agents/skills/` | ✅ 同じ場所を参照 |
| built-in image_gen | ✅ | ✅ |
| 参照画像付き編集 | ✅ | ✅（スレッドに写真を添付） |
| ローカルファイル出力 | ✅ | ✅（同じ PC 上で実行） |
| Codex cloud（リモート実行） | ⚠️ ローカルファイルシステム不可 | — |

---

## ライセンス

MIT License

---

## 関連

- [AI-exterior-planner](https://github.com/harunamitrader/AI-exterior-planner) — 外構・造園の初回提案支援ツール
- [提案ページのサンプル](https://harunamitrader.github.io/harunami_AI_base/showcase/garden-carport-proposal/)
