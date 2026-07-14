# CLAUDE.md — 掲示板(BBS) AI駆動開発

このファイルは毎セッション自動で読み込まれる恒久ルール。**常時必須のルールは本ファイルに直書き**し、詳細資料は `docs/` を参照する。全体設計は `docs/foundation-plan.md`、確定事項は `docs/decisions/` を正とする。

---

## 1. プロジェクト概要
- ローカル/限定公開・**学習用途**の掲示板(BBS)Webアプリ。
- ID/PW による**必須アカウント認証**。投稿の作成/一覧/削除、新着の即時反映（段階導入）。
- 開発方針: **TDD**、**UI/UX重視**、**技術判断の全記録**。

## 2. 技術スタック（確定 / decision 0001）
- アーキテクチャ: **分離型**（フロント/バック分離）+ リバースプロキシで同一オリジン化
- フロント: **React + Vite + TypeScript**
- バック: **Rust + Axum**（Tokio）
- DBアクセス: **SQLx**（コンパイル時SQL検証 / migration は `sqlx-cli`）※ORM不採用
- DB: **PostgreSQL**
- 認証: **Cookieセッション**（`argon2` + `tower-sessions` sqlxストア）※JWT不採用
- 即時反映: **WebSocket**（Axum）を段階導入
- 環境: **Docker + docker-compose**（frontend / backend / db / reverse-proxy）
- テスト: 前=Vitest + Testing Library + Playwright(E2E) / 後=cargo test + reqwest
- Lint/Format: 前=ESLint + Prettier / 後=cargo fmt + clippy

> スタックの変更・追加ライブラリ選定は decision 記録（§5）を伴う。

## 3. コマンド
> ⚠️ 実装スキャフォールド後に有効。未整備の段階では存在確認してから使う。
- 起動: `docker compose up -d` / 停止: `docker compose down`
- フロント: `npm test`（Vitest） / `npm run lint` / `npm run dev`
- バック: `cargo test` / `cargo clippy` / `cargo fmt` / `cargo run`
- E2E: `npx playwright test`
- マイグレーション: `sqlx migrate run` / `sqlx migrate add <name>`

## 4. 開発の進め方（常時必須）
### TDD（テストファースト厳守）
1. **Red**: 失敗するテストを先に書く
2. **Green**: 最小実装で通す
3. **Refactor**: 重複除去・整理（テストは緑のまま）
- 層: 単体（ロジック/バリデーション）→ 結合（API＋DB）→ E2E（投稿→表示）。

### Definition of Done（機能完了の基準）
1. Red→Green→Refactor を踏んだ
2. `/verify` で実挙動を確認（テスト緑だけで満足しない）
3. **UI/UXチェックリスト**（`docs/ui-ux-guidelines.md`）を満たす
4. `/security-review` を通過（入力を扱う機能は必須）
5. 独自判断は `docs/decisions/` に記録済み（§5）
6. PR作成（テンプレ充足）

### git 運用（常時必須）
- `main` 保護。機能ごとに短命ブランチ `feat/<name>`。
- 1論理変更 = 1コミット。**コミット/プッシュはユーザーの指示があるまで行わない**。
- メッセージは Conventional Commits（`feat:` `fix:` `test:` `docs:` `chore:`）＋関連 decision を参照（例 `refs #0007`）。
- 詳細は `docs/workflow.md`（作業前に参照）。

## 5. 決定記録ルール（最重要）
- **ユーザー確認なしに方針を決めた場合（＝AIの独自判断）は、実装を続ける前に必ず `docs/decisions/` に記録する。**
- 形式は `docs/decisions/TEMPLATE.md` に従い、`/log-decision` スキルを使う（連番・frontmatter・索引更新を担保）。
- frontmatter に `importance`（critical/major/minor）と `decided_by`（ai/user/ai+user）を必ず付与。**AI独自判断は `decided_by: ai`**。
- 技術判断は**すべて**残す（粒度: すべて記録し、重要度で分類）。
- `Stop` フックが記録漏れを検知して差し戻す。差し戻されたら記録するか「記録不要」と明示してから終了する。

## 6. UI/UX 必須要件
- **UI に関わる作業の前に `docs/ui-ux-guidelines.md` を読む。**
- 要点: 状態網羅（空/読込/エラー/末尾）、送信中/成功/失敗のフィードバック、クライアント＋サーバの二重バリデーション、a11y（セマンティックHTML・キーボード操作・フォーカス可視・コントラスト・label紐付け）、レスポンシブ（横スクロールさせない）。

## 7. セキュリティ必須要件
- **XSS**: 出力エスケープ／サニタイズ（BBSはユーザー入力を保存・表示するため最重要）。
- **SQLインジェクション**: SQLx のプレースホルダを使う（文字列連結でクエリを組まない）。
- **CSRF**: Cookieセッションのため状態変更操作に対策（SameSite＋CSRFトークン）。
- **認証**: パスワードは `argon2` でハッシュ。セッションは HttpOnly + Secure + SameSite。失効（ログアウト/BAN/パス変更時）を実装。
- **レート制限・スパム対策**を投稿系に入れる。

## 8. やってはいけないこと
- 本番/共有DBへの破壊的操作。
- 認証・暗号・セッションの**独自実装**（実績あるcrate/手法を使う）。
- テストの削除・スキップ・実装追従の改変で緑にすること。
- ユーザー確認なしの方針決定を**記録せずに**実装を進めること。
- 指示のない `git commit` / `git push`。
