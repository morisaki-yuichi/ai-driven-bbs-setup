# 掲示板(BBS) AI駆動開発 — 基盤構築計画書

> 本書は「実装前に、開発がスムーズかつ統制の効いた状態で進むための基盤」を設計する計画書です。
> **本書自体は実装を含みません。** 各項目は「何を・なぜ・どう構築するか」を定義し、承認後に順次構築します。

---

## 0. スコープと非スコープ

- **スコープ**: 開発の土台（規約・記録の仕組み・自動強制・ワークフロー・ツール設定）の設計。
- **非スコープ**: BBSの機能実装、UIデザインの実物、DBスキーマの確定（これらは基盤構築後の別フェーズ）。

前提として取り込む開発条件:
- **TDD**（テスト駆動）で進める
- **UI/UX・ユーザビリティ**を常に考慮する
- AIの独自判断とその根拠を**必ず文書化**する（Stopフックで強制）
- 技術判断は**すべて記録**し、**重要度で分類**する

---

## 1. 技術スタック（確定 / decision 0001）

要件ヒアリング（小規模学習用・ID/PW必須認証・即時反映ベター・ローカル/限定公開）を経て確定。詳細と代替案の却下理由は `docs/decisions/0001-tech-stack.md`（importance: critical）に記録。

| レイヤ | 採用 | 備考 |
|---|---|---|
| アーキテクチャ | **分離型**（フロント/バック分離） | API境界が明確。学習価値・Docker多コンテナと好相性 |
| フロントエンド | **React + Vite + TypeScript** | SPA。UI/UX・a11y を担当 |
| バックエンド | **Rust + Axum**（Tokio） | 型安全・並行安全・単一バイナリ |
| DBアクセス | **SQLx**（コンパイル時SQL検証） | migrationは `sqlx-cli`。ORMは不採用 |
| DB | **PostgreSQL** | Dockerコンテナで運用 |
| 認証 | **Cookieセッション**（`argon2` + `tower-sessions` sqlxストア） | 即時失効・HttpOnlyでXSS耐性。JWTは不採用 |
| 即時反映 | **WebSocket**（Axum）※段階導入 | まずリロード→後からWS |
| 実行環境 | **Docker + docker-compose** | frontend / backend / db /(reverse-proxy) |
| オリジン統合 | **リバースプロキシ**（nginx or Traefik） | 同一オリジン化しCORS回避・Cookie有効化 |
| テスト | 前: **Vitest + Testing Library + Playwright(E2E)** / 後: **cargo test + reqwest** | TDDの主戦場 |
| Lint/Format | 前: **ESLint + Prettier** / 後: **cargo fmt + clippy** | フックで自動実行 |

> 本計画の枠組み（記録・hook・ワークフロー・DoD）はスタック非依存で、上記に紐付けて具体化する。

---

## 2. リポジトリ／ディレクトリ構成

```
ai-driven-bbs-setup/
├─ CLAUDE.md                 # 恒久ルール（自動読込）
├─ docs/
│  ├─ foundation-plan.md     # 本書
│  ├─ workflow.md            # git/開発ワークフロー詳細
│  ├─ ui-ux-guidelines.md    # UI/UX・アクセシビリティ方針
│  └─ decisions/             # 決定記録(ADR)
│     ├─ README.md           # 索引＋重要度の定義
│     ├─ TEMPLATE.md         # 記録テンプレ
│     └─ 0001-*.md …         # 決定1件=1ファイル
├─ .claude/
│  ├─ settings.json          # 権限・hooks・env
│  └─ skills/
│     └─ log-decision/       # 決定記録スキル
└─ (実装は基盤構築後)
```

**まず git 化**（現在リポジトリではない）。差分確認・巻き戻し・decisions のバージョン管理の前提。

---

## 3. CLAUDE.md 構成（節立て）

自動読込される恒久ルール。以下の節を持つ。

> **読み込みの原則**：自動でコンテキストに載るのは CLAUDE.md 系のみ。`docs/` 配下は自動読込されない。
> したがって **常時効かせたいルールは CLAUDE.md に直接書くか、`@docs/xxx.md` インポート構文で取り込む**。
> 分量の多い参照資料は `docs/` に置き、CLAUDE.md からは「作業前に読む」指示＋DoD/フックで担保する。

以下の節を持つ:

1. **プロジェクト概要**：BBSの目的・機能範囲
2. **技術スタック**：着手ゲートで確定した内容
3. **コマンド**：`dev` / `test` / `test:watch` / `lint` / `e2e` / migrate 等
4. **開発の進め方（常時必須・CLAUDE.mdに集約）**：TDDサイクル、Definition of Done、コミット規約・ブランチ戦略。詳細手順は `@docs/workflow.md` をインポート（→ §7, §8, §9）
5. **決定記録ルール**：「ユーザー確認なしに方針を決めたら、実装継続前に `docs/decisions/` に記録」（→ §5）
6. **UI/UX 必須要件**：`docs/ui-ux-guidelines.md` を**参照方式**（UI作業前に読む指示）＋DoDで担保（→ §10）
7. **セキュリティ必須要件**：XSS/CSRF/SQLi対策、入力サニタイズ、レート制限、スパム対策
8. **やってはいけないこと**：本番DB操作、認証の独自実装、テスト削除で緑にする 等

---

## 4. 決定記録(ADR)の運用と重要度分類

### 4.1 方針
- **すべての技術判断を残す**。決定1件 = 1ファイル、連番 `NNNN-slug.md`。
- **重要度で分類**し、後から重要な判断だけを素早く辿れるようにする。
- **AI独自判断／ユーザー合意**を明示（要望：自律判断だけを抽出可能に）。

### 4.2 テンプレート（`docs/decisions/TEMPLATE.md`）
frontmatter で機械可読にし、索引生成や抽出を容易にする:

```markdown
---
id: 0001
title: <決定の要約>
date: 2026-07-14
importance: critical | major | minor
decided_by: ai | user | ai+user      # 自律判断は ai
status: accepted | superseded
supersedes: null
---

## 背景・問題
何を解決するためか。

## 決定
採用した選択。

## 根拠・理由
なぜそれを選んだか（AI独自判断の場合は判断の決め手を明記）。

## 検討した代替案と却下理由
- 案A：却下理由
- 案B：却下理由

## 影響範囲・見直し条件
どこに波及するか／どうなったら再検討するか。
```

### 4.3 重要度の定義（`docs/decisions/README.md` に明記）

| importance | 対象例 | 覆した時の影響 |
|---|---|---|
| **critical** | DB設計・データモデル・認証方式・セキュリティ方式 | 広範囲・高コスト |
| **major** | API設計・状態管理・主要ライブラリ選定・ディレクトリ構造 | 中程度 |
| **minor** | 命名規則・局所的UI選択・小さなリファクタ方針 | 局所的 |

README には importance 別の索引（decided_by=ai を絞り込んだ一覧を含む）を置く。

---

## 5. 決定記録スキル `/log-decision`

`.claude/skills/log-decision/` に配置。役割:
- TEMPLATE に沿って**書式のブレをなくす**（連番採番・frontmatter・重要度判定の観点を内蔵）。
- Stopフック（§6）が記録を促した際、Claude がこのスキルで一貫した形式で記録する。
- 引数で `importance` と `decided_by` を受け、README索引も更新。

> スキルはこの1つのみ初期作成。メタスキル／設計判断スキルは**作らない**（CLAUDE.md のルーブリックで代替。繰り返しが確認できたら昇格）。

---

## 6. Hook 構成（Stopブロック方式）

### 6.1 主フック：決定記録の強制（Stop）
`.claude/settings.json` の `Stop` フックに検証スクリプトを設定。ロジック:

1. Stopイベントの payload から `stop_hook_active` を確認。
   - **true なら即 exit 0（許可）** … 無限ループ防止（一度差し戻した後は止まれる）。
2. false の場合、ヒューリスティックで「記録漏れ」を検知:
   - 直近コミット以降に**実装ソースが変更**されている、かつ
   - `docs/decisions/` に**新規/更新がない**
   - → 記録漏れの疑い。
3. 疑いあり → **exit 2（ブロック）**。差し戻しメッセージで
   「今ターンに独自の技術判断があれば `/log-decision` で記録してから終了。無ければ記録不要と明示して再度終了」と促す。
4. Claude は自動続行し、記録 or 「記録不要」を確認 → 次の Stop は `stop_hook_active=true` で許可。

> ⚠️ 限界（明記済み）：hook は「判断の有無」を完全自動検知できない。上記は*ソース変更×記録なし*という代理シグナル。最終担保は CLAUDE.md 規約 + この差し戻しの合わせ技。ブロックは手動再開不要（Claudeが自動続行）。

### 6.2 補助フック（候補）
- **PostToolUse(Edit/Write)** → 変更ファイルに対し Lint/Format 自動実行（規約逸脱の早期検出）。
- **PostToolUse** → TDD補助として関連テストの自動実行（ノイズが多ければ `/verify` に一本化）。まず規約＋DoDで運用し、必要なら追加。

---

## 7. Git／開発ワークフロー（`docs/workflow.md` ＋ CLAUDE.md集約）

> **読み込み方針**：下記のうち **常時必須（ブランチ戦略・コミット粒度・メッセージ規約・コミットタイミング）は CLAUDE.md に集約**、または CLAUDE.md 先頭で `@docs/workflow.md` としてインポートし毎回コンテキストに載せる。`docs/workflow.md` は詳細・具体例の置き場とする。

- **ブランチ戦略**：`main` 保護。機能ごとに短命ブランチ `feat/<name>`（Claudeは既定でmain上なら自動分岐）。
- **コミット粒度**：1論理変更=1コミット。関連する decision は実装と同じ or 直前コミットに含める。
- **コミットメッセージ**：Conventional Commits（`feat:` `fix:` `test:` `docs:`）＋関連 decision番号を参照（例 `refs #0007`）。
- **PR内容テンプレ**：背景／変更点／関連decision／テスト結果／UI確認結果。
- **コミット/プッシュのタイミング**：Claudeは**指示があるまでコミットしない**既定。ワークフローに「機能単位でユーザー承認後コミット」を明記。

---

## 8. Definition of Done（TDD・UI/UX込み）

1機能を「完了」と見なす基準:
1. **Red**：失敗するテストを先に書いた
2. **Green**：最小実装でテストが通る
3. **Refactor**：重複除去・整理（テストは緑のまま）
4. `/verify` で実挙動を確認（テスト緑だけで満足しない）
5. **UI/UX チェックリスト**（§10）を満たす
6. `/security-review` を通過（入力を扱う機能は必須）
7. 独自判断は `docs/decisions/` に記録済み
8. PR作成（テンプレ充足）

---

## 9. TDD 方針

- **テストファースト厳守**：実装前に振る舞いをテストで定義。
- **層**：単体（ロジック/バリデーション）→ 結合（API＋DB）→ E2E（投稿→表示の主要フロー）。
- **BBS特有の必須テスト観点**：投稿の作成/一覧/削除、空入力・長文・特殊文字、XSSペイロードのエスケープ、権限（他人の投稿を消せない）、ページング。
- **禁止事項**（CLAUDE.mdにも記載）：テストを消す/スキップして緑にする、実装に合わせてテストを後付け改変する。

---

## 10. UI/UX・ユーザビリティ方針（`docs/ui-ux-guidelines.md`）

> **読み込み方針**：分量が出やすいため **`docs/` 参照方式**（自動読込しない）。CLAUDE.md に「UI作業前に `docs/ui-ux-guidelines.md` を読む」指示を置き、DoD(§8)のUIチェックで実効性を担保する。

BBSの各画面が満たすべきチェックリスト（DoDに直結）:
- **フィードバック**：送信中/成功/失敗の状態表示、エラーメッセージは具体的に。
- **状態網羅**：空状態（投稿ゼロ）、読み込み中、エラー、末尾/ページング。
- **入力体験**：クライアント側バリデーション＋サーバ側検証の二重化、文字数表示。
- **アクセシビリティ(a11y)**：セマンティックHTML、キーボード操作、フォーカス可視、コントラスト比、フォーム label 紐付け、適切な ARIA。
- **レスポンシブ**：モバイル幅で横スクロールしない。
- **検証**：Playwright(MCP) で主要フローを実ブラウザ確認。UIに関わる判断は decision に `importance` 付きで記録。

---

## 11. 権限・MCP 設定（`.claude/settings.json`）

- **permissions.allow**：`git status/diff/add/commit`、フロント `npm test`/`npm run lint`、バック `cargo test`/`cargo clippy`/`cargo fmt`、`sqlx` 系、`docker compose` 系など安全な定型を許可し確認プロンプト削減（`/fewer-permission-prompts` で精緻化）。
- **MCP**：
  - **context7**（有効）：Axum / SQLx / tower-sessions / React 等の最新ドキュメント参照。
  - **Playwright MCP**：UI/E2E の実ブラウザ検証。
- **env / hooks**：§6 のフック定義を格納。

---

## 12. セッション総括の文書化（最終フェーズ）

- 開発中は §4 の ADR を蓄積 → **最終まとめの一次資料**になる。
- 本セッション（基盤設計の議論）は最後に別途総括文書として出力予定。ADRが揃っていれば精度・省力ともに向上。

---

## 13. 構築ステップ（承認後の実行順）

1. `git init` とベースディレクトリ作成
2. 着手ゲート(§1)の確定 → 各項目を critical decision として記録
3. `docs/decisions/`（README・TEMPLATE）整備
4. `/log-decision` スキル作成
5. CLAUDE.md 作成
6. `docs/workflow.md` / `docs/ui-ux-guidelines.md` 作成
7. `.claude/settings.json`：権限 + Stopフック（+補助フック）設定
8. MCP（context7 は有効・Playwright追加）設定
9. スモーク：意図的に「記録漏れ」を起こしStopフックのブロックとログ強制を検証
10. 基盤完了 → 実装フェーズへ

---

## 14. 未決事項（次に決めるべきこと）

- 着手ゲート(§1)の各推奨を承認するか／変更するか（特に**認証方式**とスタック）。
- BBSの機能範囲（匿名投稿のみ／ログイン・スレッド・返信・画像投稿・モデレーション）。
- 補助フック（Lint自動実行・テスト自動実行）を初期から入れるか。
```
