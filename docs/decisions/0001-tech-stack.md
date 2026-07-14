---
id: 0001
title: 技術スタック選定（分離型 / React + Rust(Axum,SQLx) / PostgreSQL / Cookieセッション）
date: 2026-07-14
importance: critical
decided_by: ai+user
status: accepted
supersedes: null
---

## 背景・問題
掲示板(BBS)Webアプリを AI駆動開発で作るにあたり、実装・テスト・環境構築が依存する主要技術を確定する必要がある。

要件（ヒアリング結果）:
- 小規模・学習用途
- ID/PW による**必須アカウント**認証
- 新着の**即時反映がベター**（必須ではない）
- **ローカルで動けばよい**（限定公開・SEO不要）
- 言語候補は TypeScript / Python / Rust
- 追加条件: Docker で構築、DBは PostgreSQL、フロント/バック分離、フロントは React

## 決定
- アーキテクチャ: **分離型**（フロントエンド / バックエンド分離）
- フロントエンド: **React + Vite + TypeScript**
- バックエンド: **Rust + Axum**（Tokio）
- DBアクセス: **SQLx**（コンパイル時SQL検証、migrationは `sqlx-cli`）
- DB: **PostgreSQL**
- 認証: **Cookieセッション**（`argon2` によるパスワードハッシュ + `tower-sessions` の sqlx ストア）
- 即時反映: **WebSocket**（Axum）を**段階導入**（初期はリロード/ポーリング）
- 実行環境: **Docker + docker-compose**（frontend / backend / db /（reverse-proxy））
- オリジン統合: **リバースプロキシ**（nginx or Traefik）で同一オリジン化し CORS を回避

## 根拠・理由
- **分離型**: API境界が明確になり、API設計・認証フロー・compose 複数サービス運用を体系的に学べる。学習用途と Docker マルチコンテナに好相性。
- **Rustバックエンド**: 型安全・網羅的エラー処理が TDD 方針と噛み合う。SQLx のコンパイル時SQL検証、Axum(Tokio) の安全な非同期で WebSocket を学べる。単一バイナリで Docker イメージが極小。学習資産価値が高い。初速が最も遅い点は学習用途では許容。
- **SQLx（ORM不採用）**: 型安全に生SQLを書く学習価値と、スキーマに対する静的検証を重視。
- **PostgreSQL**: Docker 前提なら運用コストはコンテナ1つ分で、本番相当DBを学べる利点が勝る（当初検討の SQLite を撤回）。
- **Cookieセッション（JWT不採用）**: 掲示板は BAN・強制ログアウト・パスワード変更時の全セッション無効化など**即時失効**が重要で、JWTはここが弱い。HttpOnly Cookie は XSS 窃取に強く、ユーザー入力を保存・表示する BBS の攻撃面に適する。JWTの利点（ステートレス・水平スケール・多サービス）は本件では不要。

## 検討した代替案と却下理由
- **バックエンド TypeScript（Next.js等/一体型やExpress）**: 1言語で最速だが、学習軸として Rust を選択。
- **バックエンド Python（Django/FastAPI）**: Djangoは認証標準装備で実装は最も楽だが、学習軸として Rust を選択。
- **一体型（モノリス/SSRフレームワーク）**: 手間は少ないがAPI境界が暗黙。学習価値の観点で分離型を採用。
- **SQLite**: ローカル最小構成には手軽だが、Docker 前提では Postgres の手間が小さく、本番相当を学べる Postgres を採用。
- **ORM（SeaORM/Diesel）**: 生SQLの学習価値と SQLx の静的検証を優先し不採用。
- **JWT認証**: 即時失効の困難さと browser 保存時のXSS/CSRFトレードオフにより不採用。

## 影響範囲・見直し条件
- 影響: リポジトリ構成、docker-compose、テスト/Lint構成、認証実装、CI/フック、CLAUDE.md のコマンド群。
- 見直し条件: 要件が「複数サービス/水平スケール/モバイル・サードパーティAPI」へ拡大した場合は認証(JWT)・アーキテクチャを再検討。学習コスト過大でスケジュールが破綻する場合はバックエンド言語を再検討。
