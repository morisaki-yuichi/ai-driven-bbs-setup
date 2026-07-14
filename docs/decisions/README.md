# 決定記録（ADR）索引

技術判断は**すべて**この配下に「決定1件 = 1ファイル」（`NNNN-slug.md`）で記録する。
書式は `TEMPLATE.md` に従い、frontmatter で機械可読にする。

## 重要度(importance)の定義

| importance | 対象例 | 覆した時の影響 |
|---|---|---|
| **critical** | DB設計・データモデル・認証方式・セキュリティ方式・スタック選定 | 広範囲・高コスト |
| **major** | API設計・状態管理・主要ライブラリ選定・ディレクトリ構造 | 中程度 |
| **minor** | 命名規則・局所的UI選択・小さなリファクタ方針 | 局所的 |

## decided_by の定義
- `ai` … AIが開発中に独自判断（**要注目**：後から抽出できるよう必ず付与）
- `user` … ユーザーが決定
- `ai+user` … 相談のうえ合意

## 索引

| id | title | importance | decided_by | status |
|---|---|---|---|---|
| [0001](0001-tech-stack.md) | 技術スタック選定 | critical | ai+user | accepted |
