# 実装の2原則（背景・出典・詳細）

`new-project-setup-prompts.md` の実装方針に組み込んだ2つの原則について、**出典・定義・なぜ効くか・このプロジェクトでの当てはめ・注意点・さらに学ぶための参照**をまとめる。プロンプト集が「何を守るか（what）」なら、本書は「なぜそれか・どう深めるか（why / how to learn）」。

対象:
1. **Action / Calculation / Data の分離**
2. **意図の置き場所（コード=How ／ テスト=What ／ コミット=Why ／ コメント=Why-not）**

いずれも**強制の分類・法ではなく“既定の型（default heuristic）”**として採用している。単純な CRUD に過剰な層や儀式を持ち込むためのものではない。

---

## 1. Action / Calculation / Data の分離

### 1.1 出典

- **Eric Normand『Grokking Simplicity』（Manning, 2021 / 邦訳『関数型プログラミングの基礎知識』系）** … A/C/D という語彙とその分類法・段階的設計（stratified design）の直接の出典。
- **Gary Bernhardt「Functional Core, Imperative Shell」（"Boundaries" 講演, 2012 / Destroy All Software）** … 同じ発想を「純粋な核（関数型）＋薄い副作用の殻（命令型）」という構造として提示した源流。A/C/D の実装上の帰結がほぼこれ。
- 背景概念: **参照透過性（referential transparency）／純粋関数** … 同じ入力に必ず同じ出力を返し、観測可能な副作用を持たない関数。関数型プログラミング一般の基礎。

### 1.2 定義と見分け方

| 区分 | 定義 | 見分ける問い |
|---|---|---|
| **Data（データ）** | 出来事や事実を表す不活性な値。それ自体は何もしない | 「ただの値/レコードか？」 |
| **Calculation（計算）** | 入力から出力を求める**純粋関数**。副作用なし・決定的 | 「同じ入力なら**いつ・何回呼んでも**同じ結果か？」→ Yes |
| **Action（アクション）** | **呼ぶタイミングや回数が結果に影響する**操作。I/O・DB・ネットワーク・可変状態・現在時刻・乱数 | 同じ問いに対し **No**（外界に依存/影響する） |

キーになるのは Normand の「**いつ・何回呼んだかが問題になるか**」というテスト。問題になれば Action、ならなければ Calculation、呼ぶ対象ですらなければ Data。

**Action は“伝染”する**: 純粋な関数の中で1つでも Action を呼ぶと、その関数全体が Action になる（async の色付けに似た伝染）。だから Action を**呼び出しの縁（殻）へ押し出し**、中心を Calculation で厚くするのが設計目標＝functional core / imperative shell。

### 1.3 なぜ効くか（特にこのプロジェクトで）

- **テスト容易性**: Calculation は mock も DB も要らず、入力→出力を突き合わせるだけで単体テストできる。TDD の Red-Green が速く回る（`tech-selection-rationale.md` の F 軸）。
- **AI非決定性の制御**: 純粋な計算は**決定的**なので、AIが生成しても振る舞いが確定し、安価に検証できる。危険な Action（DB・セッション・HTTP＝**ブラウザ越しデバッグが最も高コスト**）を薄く隔離すれば、AIのミスが起きやすい領域が小さく・見通しよくなる。これは shift-left／非決定性制御という本プロジェクトの通奏低音と一致。
- **高コストな E2E への依存を減らす**: 純粋な核を単体テストで厚く覆うほど、agent-browser による E2E（遅い・壊れやすい）で確認すべき範囲が縮む。
- **推論しやすさ**: 参照透過な部分は局所推論できる（呼び出し文脈を追わなくてよい）。

### 1.4 このBBSでの当てはめ

- **Data**: `User` / `Thread` / `Comment` のレコード、リクエストのパラメータ、検索クエリ文字列など。
- **Calculation**: パスワード強度判定、表示名の文字数チェック、**削除可否の判定**（`thread.author_id` と現在ユーザ・コメント数 → 可否の bool）、検索フィルタの述語（`is_deleted=false`）、ソート比較関数、ページネーションの offset/limit 計算、コメント数の集計ロジック（データを与えれば純粋）、固定文言のマッピング。
- **Action**: DB の読み書き、セッションの生成/破棄、**argon2 でのハッシュ生成**（毎回ランダムソルト＝呼ぶたび結果が違う＝Action の好例）、HTTP レスポンス送出、現在時刻の取得、ID/トークンの乱数生成、ロギング。

> 「argon2 ハッシュ生成」が Action で「削除可否判定」が Calculation、という切り分けが 1.2 の問い（同じ入力で毎回同じ結果か）を実感する良い例。

### 1.5 注意（やり過ぎない）

- 型付き言語では **Data は既に型/構造体で表現済み**なので、実利の8割は **Action と Calculation の分離**（副作用の隔離）にある。そこを主眼に。
- 単純なハンドラに無理な層・間接を作らない。「**純粋な計算に寄せ、Action は薄く保つ**」既定として運用し、複雑さが利得を上回るなら素直に書く。

### 1.6 さらに学ぶ

- Eric Normand, *Grokking Simplicity*（書籍）／ eric-normand.me のブログ・動画。
- Gary Bernhardt, "Boundaries"（講演動画）, Destroy All Software の "Functional Core, Imperative Shell"。
- 参照透過性・純粋関数（関数型プログラミング入門一般）。

---

## 2. 意図の置き場所（How / What / Why / Why-not）

### 2.1 出典

この4分割の言い回し自体は、特定の一次出典というより**複数の実務家の格言が合流したコミュニティ的な定式化**。近い一次ソースは以下:

- **コミット＝Why**: Chris Beams「How to Write a Git Commit Message」（2014）… 本文は「**何を・なぜ**」を説明し「どうやって（How）」はコードに語らせる、という定番。git/Linux のコミット規約も同趣旨。
- **コメント＝Why（What/How の再説明にしない）**: Robert C. Martin『Clean Code』のコメント論、Jeff Atwood「Coding Without Comments」（2008）… 「コードが**どう**動くかはコードが語る。コメントは**なぜ**を語る」。ここから「却下した代替＝Why-not」への精緻化。
- **テスト＝What（仕様）**: Kent Beck『Test-Driven Development: By Example』（2002）、Dan North の BDD … テストを「**振る舞いの実行可能な仕様**」として書く発想。
- 深い背景: **Peter Naur「Programming as Theory Building」（1985）** … ソースコード単体では作り手の“理論（意図）”は伝わらない、という論。だからこそ意図を複数の層（コミット/コメント/決定記録）に分けて残す必要がある、という根拠になる。

### 2.2 各対応と含意

| 置き場所 | 何を書くか | 書かないこと |
|---|---|---|
| **コード** | **How**（どう実現するか） | — |
| **テスト** | **What**（期待する振る舞い＝仕様） | 実装の再現 |
| **コミット本文** | **Why**（なぜこの変更をするか・背景） | How の逐一（差分が語る） |
| **コメント** | **Why / Why-not**（非自明な理由・**却下した代替**・落とし穴） | **How の再説明**・コードの逐語訳 |

### 2.3 なぜ効くか（既存の仕組みと層状に接続する）

意図は1か所に押し込めず、**寿命と粒度で置き場所を分ける**と腐りにくい。本プロジェクトの構成にきれいに乗る:

- **大きな Why（永続・再検討対象）** → `dev-docs/decisions/`（ADR）。
- **局所の Why（この変更の動機）** → コミット本文（大きな Why は decision を参照し**重複させない**）。
- **インラインのトレードオフ（Why-not）** → コメント。
- **What（振る舞いの仕様）** → テスト。
- **How（実現手段）** → コード。

＝ decisions → commit → comment → test → code という**意図の階層**。`tech-selection-rationale.md` で組んだ decision 記録系と自然につながる。

### 2.4 注意（排他ルールにしない・緩和）

- **コメント＝Why-not は狭すぎる**。正しくは「コードが語れないこと＝**Why / Why-not**（非自明な理由・却下案）。**How の再説明は書かない**」。公開APIの doc コメント（rustdoc/JSDoc の“使い方”記述）・不変条件・警告・TODO は別枠で正当。
- **コミット＝Why** も、ヘッダは簡潔な What（Conventional Commits の要約行）＋**本文が Why**。純粋な Why 一辺倒ではない。
- **テスト＝What** は強く正しいが、テスト名・構造に What を載せ、凝りすぎて意図が読めなくならないように。
- 総じて「排他的な法」でなく「**どこに何を置くのが自然か**」の指針として運用する。

### 2.5 さらに学ぶ

- Chris Beams, "How to Write a Git Commit Message"（記事）。
- Jeff Atwood, "Coding Without Comments" / Robert C. Martin, *Clean Code*（コメント章）。
- Kent Beck, *TDD by Example* ／ Dan North, "Introducing BDD"。
- Peter Naur, "Programming as Theory Building"（1985, 論文）。

---

## 3. この2原則の位置づけ

両方とも、本プロジェクトの一貫テーマ「**意図と非決定性をどこで管理するか**」に乗っている:

- **原則1（A/C/D）** は**非決定性**を扱う: 決定的な純粋計算を核に、非決定的な副作用を殻へ。→ テスト・検証・AI生成の統制。
- **原則2（意図の置き場所）** は**意図**を扱う: How/What/Why/Why-not を寿命・粒度で分け、decision 記録系と層状に接続。

いずれも後付けの飾りでなく、TDD・shift-left・決定記録という既存の柱を実装レベルで具体化するもの。ただし繰り返すが**“既定の型”であって強制ではない**——複雑さが利得を上回る場面では素直さを優先する。
