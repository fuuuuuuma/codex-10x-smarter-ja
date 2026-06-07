# Codexを10倍賢くする方法 — 設定・運用・連携の実践ガイド

OpenAI Codex（最新のエージェント型コーディングツール。`@openai/codex` / `~/.codex` / AGENTS.md 系。2021年の旧コード補完Codexとは別物）を、設定と運用の工夫で底上げするための実践ガイドです。**読んで学ぶ解説書**であると同時に、**AIエージェントに渡してそのまま環境を再現させる仕様書**としても使えるよう、各章に「エージェント実装用」のコピペ可能な設定を載せています。

> ⚠️ **確度について**
> - 本資料の事実・コマンド・リンクは、2026年6月時点で OpenAI 公式ドキュメント（developers.openai.com）と OpenAI 公式リポジトリ（github.com/openai）を中心に裏取りしています。
> - **【公式】= developers.openai.com / github.com/openai 由来。【コミュニティ】= 個人ブログ・GitHub Issue 等。** 各章末の「出典」に区分とURLを明記しています。
> - モデル名・バージョン番号・価格・利用枠は時期で変わります。本文で固有のモデル名を断定せず「環境で有効な名前に置き換え」としている箇所は、各自で `/status` 等で確認してください。
> - 裏が取れなかった点は本文に「（未確認）」と明記し、巻末「未確認・注意事項」にまとめています。

## TL;DR（3行）

- Codexの賢さは"モデルの地力"より、文脈（AGENTS.md）・思考量（reasoning effort）・任せ方（承認/自走）・検証（/review）の設計で決まる。
- まず `/status` で現状を見て、AGENTS.md → config.toml → Skills/MCP → 検証/運用 の順に整えるだけで出力が安定する。
- 仕上げに codex-plugin-cc（OpenAI公式）で Claude Code と連携し、実装役と審査役を別AIに分けると"身内の甘さ"が減る。

---

## はじめに — 大前提と「この資料の使い方」

この資料は「Codexを10倍賢くする方法」を、手を動かしながら学べる形でまとめたものです。最初に、全体を貫く一つの考え方を共有させてください。

**Codexの賢さは、使うモデルの性能だけで決まるわけではありません。** 同じモデルでも、「何を読ませるか（文脈）」「どこまで任せるか（権限と運用設計）」「どう検証させるか（レビューの仕組み）」を整えるだけで、出力の質は大きく変わります。この資料が扱うのは、後者の「与え方・任せ方・検証のさせ方」です。新しいモデルを待たなくても、今のCodexの設定を整えるだけで、明日から出力が変わります。

ここで言葉を一つだけ噛み砕いておきます。

- **Codex（コーデックス）**: OpenAIが提供するコーディング用のAIエージェント。ターミナル（CLI）やエディタ拡張、クラウドなどから使えます。
- **AGENTS.md（エージェンツ・ドット・エムディー）**: Codexに「このプロジェクトではこう振る舞って」と恒久的に伝える指示書のファイル。
- **config.toml（コンフィグ・ドット・トムル）**: Codex本体の設定ファイル。使うモデルや承認の厳しさなどを書きます。場所は `~/.codex/config.toml`。

### まず、今の自分の設定を見る

何を変えるか考える前に、現状を一度確認する習慣をつけます。Codex CLI を起動した状態で次を打ってください。

```bash
/status
```

`/status` は「稼働中のモデル・承認ポリシー・書き込み可能なフォルダ・コンテキスト残量」を一覧で表示します（公式 slash-commands ページに記載）。ここで「今どのモデルで、どこまで自動で動けて、どこから確認を求める設定か」が分かります。本資料の各章は、ここに映る項目を一つずつ良くしていく作業だと考えてください。

### この資料の読者と、二つの使い方

読者は二層を想定しています。

- **(A) 解説動画や記事で学ぶ、初心者〜中級者の方** — 各章は専門用語をかみ砕き、最小手順から読めるように書いています。上から順に読めば、設定の全体像がつかめます。
- **(B) この資料をそのままAIエージェントに渡して、丸ごと再現させたい方** — 各章には「エージェント実装用」ブロックがあり、コピペで動くコマンドや設定（TOML・ファイル雛形）を載せています。

つまりこの資料は、人が読んで理解するためのものであると同時に、AIエージェント（CodexやClaude Code）に読ませて実装させるための仕様書でもあります。

### 最重要：AIエージェントへの渡し方

(B) の使い方を具体的に説明します。このREADME全体を Codex か Claude Code に渡し、次のような「メタプロンプト」（エージェントへの指示そのもの）を貼り付けてください。エージェントが章を順にたどりながら、設定を実装していけます。

```markdown
あなたはこのREADMEの内容に従って私の環境をセットアップする担当です。
以下を順に実行してください。

1. ~/.codex/AGENTS.md と ~/.codex/config.toml の現状を確認する。
2. この資料の各章「エージェント実装用」ブロックの設定を、章の順番どおりに適用する。
   - AGENTS.md（恒久的な指示書）を整える
   - config.toml（モデル・reasoning effort・承認モード等）を整える
   - Skills / MCP / codex-plugin-cc を導入する
3. 破壊的な変更（既存ファイルの上書き・削除、グローバル設定の書き換え）の前には、
   必ず差分を見せて私の承認を取る。
4. 各章を適用したら、何を変えたか・残課題を1〜2行で報告する。
```

エージェントに任せる場合も、上書きや削除の前に確認を取らせる一文を必ず残してください。設定ファイルは一度壊すと元に戻しづらいためです。なお、自分の手で一章ずつ進めても、結果は同じところに着地します。

### 全体の章立て（目次）

この資料は次の8章で構成します。各章は独立して読めますが、上から順に積み上げると土台から固まります。

1. **第1章 AGENTS.md — Codexの"取扱説明書"を整える** … プロジェクト固有のルールを恒久的に読ませ、毎回説明する手間をなくす。
2. **第2章 Skills（SKILL.md）— 良いやり方を"技"として持たせる** … よく使う手順を技能として定義し、必要なときに呼び出す。
3. **第3章 MCP — Codexに"手足"を生やす（3種）** … 検索・ブラウザ・GitHubなど外部ツールに接続し、できることを増やす。
4. **第4章 思考量（reasoning effort）を変えて賢さを切り替える** … 推論の深さ・モデル・思考の可視化を目的に合わせて設定する。
5. **第5章 検証の強制 — "できました詐欺"を構造で潰す** … 完了条件と `/review` で、出力を出す前に検証させる。
6. **第6章 運用設計 — 計画・自走・安全・相談の"任せ方"** … `/plan`・`/goal`・承認モード・`/side` を使い分ける。
7. **第7章 画像・スクショ投入（Appshot）— 見せて作らせる** … 画面を見せて実装やエラー解説をさせる。
8. **第8章 Claude Code × Codex の両立 — 他社AIを"審査員"に雇う** … codex-plugin-cc で実装役と審査役を分ける。

---

## 第1章 AGENTS.md — Codexの"取扱説明書"を整える

#### これは何？

AGENTS.md（エージェンツ・ドット・エムディー）は、Codex が**毎回のセッションで最初に読み込む運用ルールのファイル**です。Markdown という、見出しや箇条書きを「#」や「-」で書く軽量なテキスト形式で書きます。Claude Code を使ったことがある人なら、`CLAUDE.md` と同じ役割だと思ってください。

Codex は「セッション（一回の作業のまとまり）」を始めるたびに、このファイルを読んでから動き出します。つまり、毎回口頭で「テストは npm test で動かして」「main ブランチに直接コミットしないで」と説明し直さなくても、AGENTS.md に一度書いておけば、それが**永続的な前提**として効きます。プロジェクトの「取扱説明書」を Codex に手渡しておくイメージです。

しかも AGENTS.md は1枚だけではなく、**階層的に重ねて読まれます**。Codex は次の順で見つけたファイルを上から連結します。

1. グローバル（全プロジェクト共通）の `~/.codex/AGENTS.md`
2. git リポジトリのルートから、いま作業しているディレクトリ（cwd）まで、各階層の AGENTS.md
3. その下のサブディレクトリにある AGENTS.md

ルールは「**近いファイルほど後に読まれて勝つ**」。つまり作業中のディレクトリに近い AGENTS.md の指示が、遠い（上位の）指示を上書きします。会社全体の就業規則の上に、部署のローカルルールが乗るイメージです。

補足の仕様も押さえておきます。

- **AGENTS.override.md**: 各階層で AGENTS.md より先にチェックされる「上書き専用」ファイル。`~/.codex/AGENTS.override.md` を置くと、元の設定を消さずに一時的なグローバル上書きができます。
- **`project_doc_max_bytes`**: 読み込むサイズの上限。既定は 32 KiB（= 32768 バイト）。これを超える長大な AGENTS.md は途中までしか読まれない可能性があるので、簡潔さが大切です。
- **`project_doc_fallback_filenames`**: AGENTS.md が無いときに代わりに探すファイル名のリスト。ここに `CLAUDE.md` を指定すれば、既存の `CLAUDE.md` をそのまま流用できます。

#### なぜ"賢くなる"のか

AGENTS.md が効くのは、Codex の出力が悪くなる原因の多くが「賢さ不足」ではなく「**前提を知らないこと**」だからです。テストの動かし方、使ってよいライブラリ、触ってはいけないファイル——これらを知らないまま動くと、それらしいけれど的外れな変更が出てきます。

AGENTS.md に**実行コマンドと禁止事項を明文化**しておくと、Codex は推測ではなく事実に基づいて動けます。具体的には次のような改善が見込めます。

- セットアップ・テスト・ビルドのコマンドを先頭に書く → Codex が手探りで `make` や `yarn` を試さず、正しいコマンドを一発で打つ
- 「完了条件（Done when）」を書く → どこまでやれば終わりかが明確になり、やり残しや過剰な作業が減る
- 禁止事項とエスカレーション（判断に迷ったら止めて聞く）ルールを書く → 危険な操作や独断の暴走を抑えられる

ポイントは、**精神論ではなく実行可能な指示**を書くこと。「丁寧にコードを書いて」より「`npm run lint` を通してからコミット」のほうが、Codex は確実に従えます。

#### 使い方（手順）

発想はシンプルです。「毎回同じ説明をしている」と気づいたら、それを AGENTS.md に移すだけ。

悩み →「Codex が毎回ビルドコマンドを間違える / 触ってほしくないファイルを書き換える」
こう頼むだけ →「このプロジェクトの AGENTS.md を作って。セットアップとテストのコマンド、コーディング規約、禁止事項を入れて」

最小手順は次の通りです。

1. **雛形を作る**: Codex CLI で `/init` を実行すると、プロジェクトを見て AGENTS.md の雛形を生成してくれます。
2. **手で整える**: 生成された雛形に、実際のコマンド・規約・禁止事項を書き足します（後述のテンプレを使うと早いです）。長くしすぎず、目安 60〜300 行に収めます。
3. **読み込みを確認する**: 下記コマンドで、Codex がいまどんな指示を読んでいるかを要約させて確認します。

```bash
codex --ask-for-approval never "Summarize the current instructions."
```

意図した内容が要約に出てくれば、AGENTS.md が正しく読まれています。

#### 具体例（1つ）

小さな Next.js（React のフレームワーク）プロジェクトで、Codex に「ログインフォームのバリデーションを直して」と頼んだ場面を考えます。

before（AGENTS.md なし）

> Codex はテストの動かし方が分からず `npm test` をいきなり実行 → スクリプト未定義でエラー。さらに自動生成ファイルの `app/generated/` を勝手に書き換え、レビューで差し戻し。

after（下記の AGENTS.md を置いた後）

> Codex は冒頭の `pnpm test` を読んで一発でテストを実行。「`app/generated/` は編集しない」を読んで該当ディレクトリを避け、修正後に `pnpm lint` まで通してから「テストが緑になったので完了」と報告。

差は Codex の能力ではなく、**前提が渡っていたかどうか**だけです。

#### エージェント実装用

そのままコピペして使える汎用テンプレートです。プロジェクトルートに `AGENTS.md` という名前で置き、`<>` の箇所を実際の値に置き換えてください。

```markdown
# AGENTS.md

## Setup
- Install: `<npm install / pnpm install など>`
- Dev server: `<npm run dev など>`

## Test / Build
- Test: `<npm test / pnpm test など>`
- Lint: `<npm run lint など>`
- Build: `<npm run build など>`
- コミット前に Test と Lint を必ず通すこと。

## Conventions（規約）
- 言語 / フレームワーク: `<TypeScript + Next.js など>`
- フォーマッタ: `<Prettier など>` の設定に従う。手で整形しない。
- 命名・ディレクトリ構成は既存ファイルに合わせる。

## Done when（完了条件）
- 対象の変更が実装され、関連テストが緑（pass）になっている。
- Lint / 型チェックがエラーなく通っている。
- 不要なデバッグコード・コメントを残していない。

## 禁止（Do not）
- `<app/generated/ など自動生成ディレクトリ>` を編集しない。
- `.env` や認証情報をコミットしない。
- main ブランチへ直接コミット / force push しない。
- 指示の範囲外のファイルを「ついで」で変更しない。

## エスカレーション（迷ったら）
- 仕様が不明・破壊的変更が必要・上記の禁止に触れそうなとき → 作業を止めて確認を求める。
```

グローバル共通ルールにしたい内容（言語の指定など）は、同じ書式で `~/.codex/AGENTS.md` に置きます。既存の `CLAUDE.md` を流用したい場合は、`~/.codex/config.toml` に次を追記すると、AGENTS.md が無い階層で `CLAUDE.md` をフォールバックとして読みます。

```toml
# ~/.codex/config.toml
# AGENTS.md が無いとき CLAUDE.md を代わりに探す
project_doc_fallback_filenames = ["CLAUDE.md"]

# 読み込み上限（既定は 32768 = 32 KiB）。必要なときだけ変更する
project_doc_max_bytes = 32768
```

読み込み確認は次のコマンドで行います（承認プロンプトを出さずに要約を取得）。

```bash
codex --ask-for-approval never "Summarize the current instructions."
```

#### 出典

- Custom instructions with AGENTS.md（階層マージ順・AGENTS.override.md・`project_doc_max_bytes` 32 KiB 既定）: https://developers.openai.com/codex/guides/agents-md
- Configuration Reference（`project_doc_fallback_filenames` ほか config.toml 設定）: https://developers.openai.com/codex/config-reference
- Best practices（AGENTS.md の書き方の指針）: https://developers.openai.com/codex/learn/best-practices
- Quickstart（`/init` を含む初期セットアップ）: https://developers.openai.com/codex/quickstart

---

## 第2章 Skills（SKILL.md）— 良いやり方を"技"として持たせる

#### これは何？

「同じ作業をするたびに、毎回ゼロからやり方を説明し直している」——Codex を使っていると、こういう手間に必ずぶつかります。コードレビューの観点、コミットメッセージの書き方、PR を出す前のチェック手順。頭の中にはやり方があるのに、毎回プロンプト（AI への指示文）に書き起こすのは面倒です。

**Skill（スキル）** は、この「決まったやり方」を 1 つのファイルにまとめて、Codex に"技"として覚えさせるしくみです。レシピカードのようなものだと考えてください。一度カードを書いておけば、その作業のときだけ Codex がカードを取り出して、書いてある手順どおりに動きます。

中心になるのが **`SKILL.md`** というファイル 1 枚です。先頭に **YAML フロントマター**（ファイル冒頭の `---` で囲んだ設定欄。`name` と `description` を書く）を置き、その下に本文として手順を書きます。スキルは 1 つのフォルダにまとまり、必要なら次のものを一緒に置けます。

- `scripts/` — スキルが呼び出す実行スクリプト
- `references/` — 参照用の追加資料（長い仕様書など）
- `assets/` — テンプレートや画像などの素材

呼び出し方は 2 通りあります。**明示呼び出し**（あなたがスキル名を指定して起動する）と、**暗黙呼び出し**（`description` の内容を見て Codex が「今これが要る」と自動で選ぶ）です。後者を効かせるには、`description` に「どんなときに使うか」を具体的に書くことが大事になります。

仕組みのキモが **Progressive disclosure（段階的開示）** です。起動した時点では Codex は `name` と `description` だけを読み、本文や `scripts/` の中身は必要になってから読み込みます。これでコンテキスト（AI が一度に読み込める作業メモリ）の無駄づかいを防ぎます。

そしてもう一つ、この SKILL.md は **Anthropic が提唱した Agent Skills**（同じ `SKILL.md` という形式）と共通の発想で作られています。そのため Claude Code 向けに書いたスキルを、ほぼそのまま Codex でも流用できるという声もあります。ただし、両者の完全な互換性をOpenAIが公式に明言した記述は確認できていません（未確認）。流用する場合は、手元で実際に動くかを確かめてから使ってください。

なお、OpenAI は従来の **カスタムプロンプト**（`~/.codex/prompts/` 配下の Markdown をスラッシュコマンド化する旧仕様）を **deprecated（非推奨）** とし、共有や暗黙呼び出しが必要なケースでは Skills への移行を案内しています（公式 custom-prompts ページに「Custom prompts are deprecated. Use skills...」と明記）。これから手順を"技"として持たせるなら、Skills が標準の置き場所です。

#### なぜ"賢くなる"のか

スキルが効くのは、Codex の出力のブレを減らすからです。プロンプトを毎回手書きすると、その日の書き方しだいで観点が抜けたり、順番が変わったりします。SKILL.md に手順を固定しておけば、Codex は毎回同じ基準で動きます。レビューなら「見るべき観点を毎回もれなく通す」、コミットなら「フォーマットを毎回そろえる」といった再現性が手に入ります。

さらに、暗黙呼び出しと段階的開示の組み合わせで、必要なときだけ必要な手順が読み込まれます。常に長い指示文を抱えさせるのではなく、その作業の局面に入った瞬間に詳しい手順が展開されるため、関係ない情報でコンテキストを圧迫せずに済みます。結果として、Codex は「いつ・何を・どの手順で」やるかを安定して判断しやすくなります。

#### 使い方（手順）

「PR を出す前のレビュー観点を毎回 Codex に説明している」という悩みがあるとします。発想はシンプルで、その説明文を 1 回だけ SKILL.md に書き写し、所定のフォルダに置くだけです。

最小の手順は次のとおりです。

1. スキル用フォルダを作る。グローバル（全プロジェクト共通）なら `~/.codex/skills/<スキル名>/` を用意します。
2. その中に `SKILL.md` を 1 枚置く。先頭に `name` と `description` を書き、本文に手順を書きます。
3. Codex を再起動する（起動時にスキル一覧を読み込むため）。
4. 試す。明示呼び出しならスキル名を指定し、暗黙呼び出しなら `description` に書いた作業をそのまま頼みます。

最初は本文を欲張らず、観点を箇条書きで数行だけにするのがコツです。動くことを確認してから、`references/` に詳細を逃がしたり `scripts/` を足したりして育てます。

#### 具体例（1つ）

「PR を出す前のセルフレビュー」を `pr-review` という最小スキル 1 枚にする例です。

Before（毎回プロンプトに手打ち）

> このディフを見て。変な命名がないか、テストの抜けがないか、エラーハンドリングが雑になってないか、コミットを分けるべきか、観点を1つずつ挙げてレビューして。あ、あとセキュリティも見て。

——この指示を毎回書くため、ある日は「セキュリティ」を書き忘れ、ある日は順番が変わります。

After（スキル化して呼ぶだけ）

> /pr-review

`description` に「PR/コミット前のセルフレビュー時に使う」と書いておけば、「この変更、出す前に見ておいて」と頼むだけで暗黙的に起動させることもできます。観点は SKILL.md に固定されているので、毎回同じ基準でレビューが返ってきます。

#### エージェント実装用

最小の `SKILL.md` 雛形です。`~/.codex/skills/pr-review/SKILL.md` として保存します。

```markdown
---
name: pr-review
description: PR やコミットを出す前のセルフレビューに使う。差分（diff）に対し、命名・テストの抜け・エラーハンドリング・コミット分割・セキュリティの観点でレビューするとき。
---

# PR セルフレビュー

現在の作業ツリーの差分（git diff）を読み、以下の観点で1つずつ指摘する。
問題がなければ「問題なし」と明記する。推測ではなく差分の根拠を引用すること。

## 観点
1. 命名: 変数・関数名が意図を表しているか。略語・タイプミスはないか。
2. テスト: 追加・変更した挙動に対応するテストがあるか。境界値の抜けはないか。
3. エラーハンドリング: 失敗パスを握りつぶしていないか。例外・戻り値の扱いは妥当か。
4. コミット分割: 無関係な変更が混ざっていないか。分けるべき単位はないか。
5. セキュリティ: 秘匿情報のハードコード、入力値の未検証、権限の緩みがないか。

## 出力形式
- 観点ごとに「指摘」または「問題なし」を返す。
- 重大度（高 / 中 / 低）を各指摘に付ける。
```

配置とディレクトリ構成（グローバルに置く場合）。

```bash
# スキル用フォルダを作成
mkdir -p ~/.codex/skills/pr-review

# 上記の SKILL.md を配置（エディタで作成 or 既存ファイルをコピー）
#   ~/.codex/skills/pr-review/SKILL.md

# 任意の補助ディレクトリ（必要になってから追加でよい）
mkdir -p ~/.codex/skills/pr-review/scripts
mkdir -p ~/.codex/skills/pr-review/references
mkdir -p ~/.codex/skills/pr-review/assets

# Codex を再起動してスキルを読み込ませる（起動時に name/description を読む）
```

完成後のレイアウトは次のようになります。

```text
~/.codex/skills/
└── pr-review/
    ├── SKILL.md          # name/description + 本文（必須）
    ├── scripts/          # 呼び出すスクリプト（任意）
    ├── references/       # 参照用の追加資料（任意）
    └── assets/           # テンプレや素材（任意）
```

補足として、スキルには任意で `agents/openai.yaml` を置き、UI 上のメタデータ・呼び出しポリシー・依存ツールを指定できます（公式 Skills ページに記載）。最初は不要なので、まず `SKILL.md` 1 枚から始めてください。

#### 出典

- Agent Skills（Codex 公式・SKILL.md の構造と `agents/openai.yaml`）: https://developers.openai.com/codex/skills
- Custom instructions with AGENTS.md（公式）: https://developers.openai.com/codex/guides/agents-md
- カスタムプロンプトの deprecated 告知（公式・「Custom prompts are deprecated. Use skills...」）: https://developers.openai.com/codex/custom-prompts
- Configuration Reference（公式・`~/.codex` 周りの設定）: https://developers.openai.com/codex/config-reference
- Grill Me Skill（Agent Skills の実例・Matt Pocock）: https://github.com/mattpocock/skills
- My Grill Me Skill Went Viral（解説・aihero.dev）: https://www.aihero.dev/my-grill-me-skill-has-gone-viral

---

## 第3章 MCP — Codexに"手足"を生やす（3種）

#### これは何？

MCP（Model Context Protocol）は、AIエージェントに外部ツールをつなぐための共通規格です。難しく聞こえますが、発想は単純で「Codexに手足を生やす配線ルール」だと思ってください。

Codex本体は、文章を読み書きしたりコードを考えたりするのは得意ですが、それだけだと「頭の中」で完結しています。最新の公式ドキュメントを読みに行く、実際のブラウザを操作してサイトを開く、GitHubのIssueやPull Request（変更提案）を触る——こうした「外の世界に手を伸ばす」動作は、単体ではできません。

MCPは、こうした外部ツールを「MCPサーバー」という小さなプログラムとしてCodexにつなぐ仕組みです。Codexは「MCPクライアント」として接続し、相手が何をできるか（ツールの一覧）を受け取り、必要に応じて呼び出します。コンセント（規格）が共通なので、対応した道具なら同じやり方で差し込めるのが利点です。接続状況は対話画面で `/mcp` と打てば確認できます（公式ドキュメント記載）。

#### なぜ"賢くなる"のか

Codexが間違える原因の多くは「知らないこと・見えないことを、知っているフリで埋める」ことにあります。MCPはこの穴を物理的に塞ぎます。

- 最新情報を知らない → ドキュメント参照ツール（Context7）で、その場で正しいAPIの書き方を読み込む。記憶頼みの古い書き方（幻覚）が減ります。
- 動かして確かめられない → ブラウザ操作ツール（Playwright）で実際にページを開き、結果を見てから直す。「たぶん動く」を「動いた／動かない」に変えられます。
- 手作業の往復が多い → GitHub操作ツールで、IssueやPRの読み書きをCodex側から行える。人がコピペで橋渡しする手間が減ります。

つまりMCPは、Codexの「賢さ」そのものを上げるというより、判断の材料と実行の手段を増やすことで、結果として出力の正確さと完了率を底上げします。

#### 使い方（手順）

まず発想から。「最新ライブラリの正しい書き方が分からなくて、Codexが古いコードを書いてくる」という悩みがあるとします。これはドキュメント参照ツールを1つ足すだけで多くが解決します。最小手順は次の通りです。

1. Context7を追加する（下のコマンドを1行実行）。
2. Codexを再起動し、対話画面で `/mcp` と打って接続を確認する。
3. 普段の頼み方の末尾に「Context7で最新版を確認してから書いて」と一言添える。

これだけで、Codexは記憶ではなく実際のドキュメントを引いてから書くようになります。ブラウザ操作（Playwright）やGitHub操作も、追加コマンドを実行して `/mcp` で確認、という流れは同じです。最初から3つ全部入れる必要はなく、いま困っていることに対応する1つから始めるのが安全です。

#### 具体例（1つ）

シナリオ: あるライブラリの新しい書き方でコードを書いてほしい、というケース。

- before（MCPなし）: 「このライブラリで設定を書いて」と頼むと、Codexは学習当時の記憶で書く。バージョンが進んで関数名が変わっていると、存在しない引数や古い書式が混ざり、動かない。

- after（Context7あり）: 「Context7で最新版のドキュメントを確認してから、このライブラリの設定を書いて」と頼む。Codexはまずドキュメント参照ツールを呼び、現行の正しい関数名・引数を読み込んでからコードを書く。記憶と現実のズレが減る。

ポイントは、ツールを入れただけでなく「確認してから書いて」と明示することです。MCPは選択肢を増やしますが、いつ使うかの指示があると外しにくくなります。

#### エージェント実装用

以下は3種それぞれの追加コマンドと、`~/.codex/config.toml` の `mcp_servers` 記述例です。パッケージ名・エンドポイントは公式リポジトリで確認済みのものを使っています。

CLIで追加する場合（`--` の後ろが起動コマンド）:

```bash
# 1) Context7（最新ドキュメント参照 / stdio型）
codex mcp add context7 -- npx -y @upstash/context7-mcp

# 2) Playwright（実ブラウザ操作 / stdio型・Microsoft公式）
codex mcp add playwright -- npx @playwright/mcp@latest

# 3) GitHub公式MCP（Issue/PR操作 / リモートHTTP型）
#    事前に環境変数 GITHUB_PERSONAL_ACCESS_TOKEN を設定しておく
codex mcp add github --env GITHUB_PERSONAL_ACCESS_TOKEN=$GITHUB_PERSONAL_ACCESS_TOKEN \
  -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

`~/.codex/config.toml` を直接編集する場合:

```toml
# Context7（stdio型）
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
# Context7のホスト版を使う場合は env で APIキーを渡す（任意）
# env = { CONTEXT7_API_KEY = "..." }

# Playwright（stdio型 / Microsoft公式）
[mcp_servers.playwright]
command = "npx"
args = ["@playwright/mcp@latest"]

# GitHub公式MCP・ローカルDocker版（stdio型）
[mcp_servers.github]
command = "docker"
args = ["run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "ghcr.io/github/github-mcp-server"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxx" }

# GitHub公式MCP・リモートHTTP版（HTTP型 / Dockerを使わない場合）
[mcp_servers.github_remote]
url = "https://api.githubcopilot.com/mcp/"
bearer_token_env_var = "GITHUB_PERSONAL_ACCESS_TOKEN"
```

確認: Codexの対話画面で `/mcp` と打つと、接続済みサーバーと利用可能なツールが一覧表示されます（`codex mcp list` 相当の専用コマンドは公式ドキュメントに明記なし＝未確認。確認は `/mcp` を使う）。

注意（最小権限と安全運用）:

- 入れすぎない。MCPサーバーが多いほどツール定義が文脈（コンテキスト）を圧迫し、かえって判断がぶれます。常用する1〜3個に絞る。
- 出所の確かなものだけ。ここで挙げた3つは、Context7=Upstash提供（`@upstash/context7-mcp`）、Playwright=Microsoft公式（`@playwright/mcp`）、GitHub=GitHub公式（`ghcr.io/github/github-mcp-server`）と確認済み。素性の不明なサーバーは入れない。
- プロンプトインジェクション／ツール毒に注意。外部サイトやIssue本文に「この指示に従え」と仕込まれていると、ツール経由でCodexが誘導される恐れがあります。トークンは最小権限で発行し、承認モード（`/permissions`）で書き込み系の動作には確認を挟む運用と併用してください。

#### 出典

- Model Context Protocol — Codex公式ドキュメント: https://developers.openai.com/codex/mcp
- スラッシュコマンド一覧（`/mcp`・`/permissions` の記載）: https://developers.openai.com/codex/cli/slash-commands
- Context7 MCP（`@upstash/context7-mcp`・エンドポイント `https://mcp.context7.com/mcp`）: https://github.com/upstash/context7
- Playwright MCP（`@playwright/mcp`・Microsoft公式）: https://github.com/microsoft/playwright-mcp
- GitHub公式MCPサーバー（リモート `https://api.githubcopilot.com/mcp/`・Docker `ghcr.io/github/github-mcp-server`・`GITHUB_PERSONAL_ACCESS_TOKEN`）: https://github.com/github/github-mcp-server

---

## 第4章 思考量（reasoning effort）を変えて賢さを切り替える

#### これは何？

Codex には「どれくらい深く考えてから答えるか」を決めるダイヤルがあります。これが `model_reasoning_effort`（モデルの推論努力＝思考量）です。reasoning（リーズニング）とは、モデルが答えを出す前に内部で行う「下書きの思考」のこと。この量を増やすほど、いきなり書き始めるのではなく、手順を整理し、見落としを潰してから出力するようになります。

取りうる値は次の5段階です（公式の Configuration Reference より）。

- `minimal`（最小）
- `low`（低）
- `medium`（中・既定相当）
- `high`（高）
- `xhigh`（最高）

公式には注記があり、`xhigh` は「Responses API のみ／モデル依存」です。つまり使っているモデルや接続方法によっては `xhigh` が効かないことがあります。設定はたった1行で、`~/.codex/config.toml` に `model_reasoning_effort = "high"` と書くだけで、Codex 全体の思考の深さが変わります。

あわせて押さえたいのが、思考の「中身」を見せる2つの設定です。

- `model_reasoning_summary`（思考の要約をどれくらい出すか）。値は `auto` / `concise`（簡潔）/ `detailed`（詳細）/ `none`（出さない）。
- `show_raw_agent_reasoning`（生の思考をそのまま表示するか）。`true` / `false` の真偽値です。

#### なぜ"賢くなる"のか

思考量を上げると、Codex は答える前に「前提の確認・複数案の比較・反例の検討」を内部で多く回します。結果として、設計判断やバグの根本原因の特定など、一発で書き切れないタスクで詰めの甘さが減りやすくなります。逆に `minimal` / `low` は思考を切り詰めるぶん速く返ってきますが、込み入った問題では浅い答えになりがちです。

`model_reasoning_summary` を `detailed` にし、必要なら `show_raw_agent_reasoning` も有効にすると、Codex が「何をどう考えたか」が画面に出ます。これには2つの利点があります。1つは、解説動画で思考プロセスそのものを見せられること。もう1つは、出力が間違っていても、その手前の思考の段階で「前提を取り違えている」と気づけることです。答えだけ見て判断するより、誤りに早く手を打てます。

#### 使い方（手順）

悩み →「簡単な質問なのに毎回じっくり考えて遅い」「逆に、重い設計判断なのに浅い答えしか返ってこない」。そんなときは、タスクに合わせて思考量を切り替えるだけです。

1. まずその場で試す。TUI（Codex の対話画面）で `/model` と打つと、モデルと思考量を選ぶポップアップが出ます。そこで `high` などを選びます。
2. 起動時に上書きする。`codex -c model_reasoning_effort="high"` のように `-c`（config 上書き）を付けて起動すると、そのセッションだけ思考量を変えられます。
3. 現在値を確認する。`/status` を打つと、稼働中のモデルや承認ポリシーなどセッション設定が表示されます。切り替えの前後でここを見比べると、いま何で動いているかが分かります。

使い分けの目安は次の通りです。

- 日常の対話コーディング（小さな修正・質問・定型作業）→ `medium`。速さと質のバランスが良く、まずはここで十分です。
- 難しいリファクタリング・設計判断・セキュリティに関わる確認 → `high`、それでも足りなければ `xhigh`。時間はかかりますが、詰めの甘さを減らせます。

毎回そのタスクだけ深く考えてほしいなら 2 の `-c` 上書きが手軽です。プロジェクト全体で常に深く考えてほしいなら、後述の `config.toml` に書いて固定します。

#### 具体例（1つ）

「ログイン後にときどきセッションが切れる」というバグ修正を、同じ依頼文で `low` と `high` に投げて比べます。

Before（`codex -c model_reasoning_effort="low"`）
返ってきたのは「セッションの有効期限が短いようです。`maxAge` を延ばしてください」という1点だけの回答。確かに直りそうですが、なぜ「ときどき」なのかには触れていません。延ばしても再発する可能性が残ります。

After（`codex -c model_reasoning_effort="high"`）
同じ依頼に対し、`model_reasoning_summary = "detailed"` で思考の要約も表示。Codex は「『ときどき』という条件から、固定の期限切れではなく、サーバーを複数台で動かしたときにセッション保存先が共有されていない可能性」「Cookie の属性（`SameSite` など）の不一致」「時刻のずれ」を順に検討し、まず「セッションの保存先がインメモリで、再起動やロードバランス時に消えていないか」を確認するよう促しました。原因の切り分けまで踏み込んでいる点が違いです。

同じモデル・同じ依頼でも、思考量を変えるだけで回答の深さが変わることが分かります。なお、これは挙動の傾向を示す例であり、毎回まったく同じ結果になるとは限りません。

#### エージェント実装用

`~/.codex/config.toml` に以下を追記します（プロジェクト全体の既定値を固定する場合）。

```toml
# モデルと思考量（reasoning effort）
model = "gpt-5-codex"            # 利用環境で有効なモデル名に置き換える（未確認: 環境ごとに異なる）
model_reasoning_effort = "high"  # minimal / low / medium / high / xhigh のいずれか

# 思考の中身を見せる設定
model_reasoning_summary = "detailed"  # auto / concise / detailed / none
show_raw_agent_reasoning = true       # 生の思考をそのまま表示する（true / false）
```

注記:

- `model` の値は利用環境で有効なモデル名に置き換えてください。具体的な既定モデル名は環境・バージョンに依存するため、ここでは固定値を断定しません（未確認）。
- `xhigh` は公式注記により「Responses API のみ／モデル依存」です。効かない場合は `high` を使ってください。

セッション単位で一時的に切り替えるコマンド（config.toml を書き換えずに上書き）。

```bash
# このセッションだけ思考量を high にして起動
codex -c model_reasoning_effort="high"

# 思考量に加えて思考の要約も詳細表示で起動
codex -c model_reasoning_effort="high" -c model_reasoning_summary="detailed"
```

TUI 内での操作（対話中に切り替え・確認）。

```markdown
/model    … モデルと思考量をポップアップから選ぶ
/status   … 稼働中のモデル・承認ポリシー・コンテキスト容量などセッション設定を表示
```

運用のコツ: `config.toml` の既定は `medium` のままにしておき、重いタスクのときだけ `/model` か `-c model_reasoning_effort="high"` で引き上げる。切り替え後は `/status` で現在値を確認してから本番の依頼を投げると、想定外の設定で走らせずに済みます。

#### 出典

- Configuration Reference（`model_reasoning_effort` の取りうる値 minimal/low/medium/high/xhigh、`model_reasoning_summary`、`show_raw_agent_reasoning` を記載）: https://developers.openai.com/codex/config-reference
- Codex CLI スラッシュコマンド一覧（`/model`＝稼働モデルの選択、`/status`＝セッション設定とトークン使用量の表示）: https://developers.openai.com/codex/cli/slash-commands
- Config basics（基本的な設定の書き方）: https://developers.openai.com/codex/config-basic

---

## 第5章 検証の強制 — "できました詐欺"を構造で潰す

#### これは何？

AI エージェントに作業を頼んだとき、いちばん多い失敗は「直しました」「動きます」と返ってくるのに、実際に動かすと壊れている、というものだ。本人（エージェント）は嘘をつくつもりがなく、コードを書いた時点で満足して報告してしまう。これを本章では「できました詐欺」と呼ぶ。

これを防ぐ考え方が「検証の強制」だ。やることは2つに分かれる。

1つ目は **Done when（完了条件）** をプロンプトや AGENTS.md に明文化すること。「完了条件」とは「何が起きたらこのタスクは終わりと見なすか」を具体的に書いた1行だ。たとえば「`pnpm test` が緑になったら完了」のように、テストの合否やコマンドの成功で判定できる形にする。`pnpm` は Node.js のパッケージ管理ツールの1つ、「緑になる」はテストが全部通った状態を指す言い回しだ。AGENTS.md は Codex が作業前に読む指示書ファイルで、ここに検証ルールを書くと毎回効く（詳細は AGENTS.md の章を参照）。

2つ目は **レビューの自動化**。Codex には `/review` というコマンドがあり、これは「読み取り専用」で別のレビュー役が作業内容を点検する。読み取り専用とは、ファイルを書き換えずに見るだけ、という意味。挙動の変化（退行＝今まで動いていた部分が壊れること）やテストの不足を見つけて指摘する。GitHub 上では `@codex review` でプルリクエスト（コードの変更提案）を自動レビューさせられる。

つまり「自分で完了条件を満たすまで直させる（指示）」と「別の目で点検させる（自動）」の両輪で、報告の正しさを構造的に担保する。

#### なぜ"賢くなる"のか

モデルそのものの賢さは変わらない。変わるのは「いつ報告してよいか」の基準だ。完了条件がないと、エージェントはコードを書き終えた瞬間をゴールと見なす。完了条件があると、ゴールが「テストが通った状態」に移動する。結果として、エージェント自身がテストを回し、失敗を読み、直し、また回す、というループを自走するようになる。人間が後から「動かないよ」と差し戻す往復が減る。

`/review` を足すと、書いた本人とは別の視点で退行やテスト欠落を重大度順に洗い出せる。1人で書いて1人で OK を出す状態から、書き手と点検役を分けた状態に変わる。

#### 使い方（手順）

悩み: 「直したと言うのに動かない」。こう頼むだけ。

1. タスク依頼の最後に完了条件を1行足す。例: 「`pnpm test` が緑になるまで、報告しないで」。
2. Codex がテストを実行し、失敗があれば自分で直して再実行する。緑になるまで報告は来ない。
3. 仕上げに `/review` を打つ。書き換えずに、退行・バグ・テスト不足を重大度順で出してくれる。
4. GitHub の PR では本文かコメントに `@codex review` と書くと、自動でレビューが付く（GitHub 連携の設定が前提）。

毎回書くのが面倒なら、完了条件を AGENTS.md に常設ルールとして書いておく。以降のタスクに自動で効く。

#### 具体例（1つ）

ログイン処理のバグ修正を頼んだ場面。

Before（完了条件なし）

> 依頼: 「ログインのバグを直して」
> 返答: 「修正しました。`auth.ts` の条件分岐を直しています。」
> → 実際に動かすと別のテストが落ちていた。差し戻し。

After（完了条件あり）

> 依頼: 「ログインのバグを直して。`pnpm test` が緑になるまで報告しないで。落ちたテストは自分で直して。」
> 返答（数分後）: 「`pnpm test` を実行 → 3件失敗 → `auth.ts` と `session.ts` を修正 → 再実行で全28件パス。完了条件を満たしました。」

報告が来た時点でテストが通っている状態が保証される。差し戻しの往復が1回で済む。

#### エージェント実装用

完了条件入りプロンプト雛形:

```markdown
# タスク
<やってほしいこと>

# 完了条件（Done when）
- `pnpm test` がすべて緑になること
- `pnpm lint` がエラー0で通ること
- 上記2つを自分で実行・確認してから報告すること
- 確認できるまで「完了」と言わないこと。失敗したテストは自分で直して再実行する
```

AGENTS.md に常設する検証ルールの例:

```markdown
## 検証ルール（Verification）
- 完了報告の前に、必ずテストとリントを実行して結果を本文に貼る
- 推測で「動くはず」と書かない。実行して確認した事実だけを書く
- テストが緑になるまでタスクを完了扱いにしない
- 変更がふるまいに影響する場合、対応するテストの追加・更新も行う
```

ローカルでの読み取り専用レビュー（Codex CLI のスラッシュコマンド）:

```bash
# 作業ツリー（未コミットの変更）をレビュー。書き換えはしない
/review

# 特定ブランチとの差分をレビューしたいとき（プラグイン版）
/codex:review --base main
```

`/review` は Codex CLI 標準のコマンドで、作業ツリーを点検し、挙動変更や不足テストに着目して指摘する（`review_model` 未設定時は現セッションのモデルを使用）。Claude Code から Codex を呼ぶプラグイン（openai/codex-plugin-cc）を入れている場合は `/codex:review`、実装判断に踏み込んで反論させたいときは `/codex:adversarial-review` が使える。

GitHub での PR 自動レビュー:

```markdown
<!-- PR 本文またはコメントに記述。AGENTS.md の Review ガイドに従ってレビューされる -->
@codex review
```

プラグイン経由でレビューをゲート（合格するまで先に進めない関門）にしたい場合:

```bash
# レビューゲートを有効化 / 無効化
/codex:setup --enable-review-gate
/codex:setup --disable-review-gate
```

#### 出典

- `/review`（作業ツリーのレビュー、挙動変更・不足テストに着目、`review_model` 未設定時は現セッションのモデル使用） — Codex CLI スラッシュコマンド一覧: https://developers.openai.com/codex/cli/slash-commands
- AGENTS.md によるカスタム指示（検証ルールの常設先） — https://developers.openai.com/codex/guides/agents-md
- ベストプラクティス（完了条件・検証の考え方の補助資料） — https://developers.openai.com/codex/learn/best-practices
- GitHub PR レビュー（`@codex review` の利用） — https://developers.openai.com/codex/use-cases/github-code-reviews
- Codex プラグイン for Claude Code（`/codex:review` `/codex:adversarial-review` `/codex:setup --enable-review-gate` 等。description=「Use Codex from Claude Code to review code or delegate tasks.」、ライセンス Apache-2.0） — https://github.com/openai/codex-plugin-cc
- Non-interactive mode（CI で `codex exec` を使い検証を自動化する補助） — https://developers.openai.com/codex/noninteractive

---

## 第6章 運用設計 — 計画・自走・安全・相談の"任せ方"

#### これは何？

Codex に仕事を頼むとき、いきなり「これ作って」と投げるのと、段取りを決めてから渡すのとでは出力が変わる。この章は「任せ方」を 4 段階で設計する話。具体的には次の 4 つの道具を使い分ける。

- 計画(プランニング): 動き出す前にゴールと制約を固める。ここで使うのが Grill Me(自己詰問インタビュー。Codex 側が「これはこういう仕様で合っていますか？」と推奨回答付きの質問を浴びせ、あなたは Yes/No で答えるだけで、抜けや矛盾を炙り出す手法)と /plan(計画モード。実装に入る前に方針を一枚にまとめる公式コマンド)。
- 自走(オートラン): 一度ゴールを決めたら、達成するか上限に当たるまで Codex が複数ターンを自分で回し続ける。これが /goal(ゴールモード)。
- 安全(権限管理): Codex がファイルを書き換えたりコマンドを実行したりする範囲を、承認ポリシー(approval_policy。実行前に許可を求めるかの設定)とサンドボックス(sandbox_mode。書き込んでよい範囲の隔離設定)で線引きする。
- 相談(サイドチャット): 本線の会話(メインのトランスクリプト=作業ログ)を汚さずに、脇で軽く質問する /side(別名 /btw)。

「ターン」とは Codex が一回考えて一回動く単位、「トークン」は文章量に応じて消費する利用量の単位、とだけ押さえておけば読み進められる。

#### なぜ"賢くなる"のか

賢さは「正しい前提」と「適切な自由度」から出てくる。Grill Me と /plan は前提のズレを着手前に潰す。人間は仕様を曖昧に渡しがちで、Codex はその曖昧さを勝手な仮定で埋めてしまう。質問で先に詰めておけば、出戻りが減る。

/goal は逆に自由度を与える側だ。長い作業を一問一答で逐一指示すると、こちらの集中も切れるし指示の質も落ちる。ゴールだけ与えて自走させ、Codex 自身に「まだ終わっていない/脱線していないか」を点検させると、人間の介入回数を減らせる。

安全設定は賢さの土台になる。権限を絞れば、Codex は危険な操作の前で必ず立ち止まる。逆に信頼できる作業では緩めて速度を上げる。OpenAI の方針は「厳しめを既定にして、必要なときだけ緩める」。これにより「勝手に消された」「想定外のコマンドが走った」を避けつつ、任せられる範囲だけ手を離せる。

#### 使い方（手順）

悩み: 「ふわっとした依頼を投げて、出来上がりが思っていたものと違う」。こう頼むだけで変わる。

1. 着手前に詰めてもらう。実装を頼む前に「実装に入る前に、要件の抜けや矛盾を質問で洗い出して」と一言添える(Grill Me を入れているなら `/grill-me`)。推奨回答付きで質問が来るので Yes/No 中心に答える。
2. 計画を固める。`/plan` と打って計画モードに入り、ゴール・前提・制約・完了条件を一枚にまとめてもらう。内容に納得してから実装へ移る。
3. 長い作業は自走させる。`/goal <達成したいこと>` でゴールを設定すると、達成または上限まで Codex が回り続ける(事前に有効化が必要。後述)。`/goal pause`・`/goal resume`・`/goal clear` で停止・再開・解除できる。
4. 権限はその場で切り替える。あらかじめ厳しめにしておき、緩めたいときだけ `/permissions` で承認プリセット(Read Only / Auto など)を切り替える。
5. 脇道の相談は本線を汚さない。`/side <聞きたいこと>` で一時的な別会話を開き、本線の作業ログを乱さずに確認する。

#### 具体例（1つ）

シナリオ: 既存の API に入力バリデーションを追加したい。

before(丸投げ):

```text
> APIにバリデーション足しといて
```

Codex は対象エンドポイントもエラーの返し方も自分で仮定し、後から「そこじゃない」「形式が違う」と何度も直すことになる。

after(計画 → 自走):

```text
> 実装前に、対象エンドポイント・必須項目・エラー応答形式について、抜けや矛盾を質問で洗い出して
（Codex が推奨回答付きで質問。Yes/No で回答し前提が固まる）

> /plan ユーザー登録APIに入力バリデーションを追加する
（ゴール・対象ファイル・完了条件＝「全項目に検証」「異常系テストが通る」が一枚にまとまる）

> /goal バリデーションを実装し、異常系テストがすべて通る状態にする
（達成条件を満たすまで Codex が実装→テスト→修正を自分で繰り返す。
 ニセ完了や脱線は continuation の点検ロジックが抑える）
```

前提を先に固め、完了条件を渡して自走させたことで、人間の指示は 3 行になり、出戻りが減る。

#### エージェント実装用

Goal モードの有効化(`~/.codex/config.toml` に追記。または CLI で `codex features enable goals` を実行し再起動)。`/goal` は実験的(experimental)機能で、Codex CLI 0.128.0 以上が必要。

```toml
# ~/.codex/config.toml
[features]
goals = true
```

承認ポリシーとサンドボックスの既定を厳しめにし、必要時だけ緩める設定例。値の正確な取り得る範囲は config-reference を参照(下記出典)。

```toml
# ~/.codex/config.toml

# 実行前に許可を求めるか（厳しめ既定）
approval_policy = "untrusted"

# 書き込み範囲の隔離: read-only / workspace-write / danger-full-access
sandbox_mode = "workspace-write"
```

セッション中の切り替え(スラッシュコマンド)。

```text
/permissions      # Codex が確認なしでできる範囲を切り替える（Auto / Read Only 等）
/status           # 現在のモデル・承認ポリシー・書き込み可能ルート・コンテキスト残量を確認
/plan <prompt>    # 計画モードに入る（タスク実行中は一時的に使えない）
/goal <objective> # ゴール設定。/goal で確認、pause/resume/clear で制御
/side <prompt>    # 本線を汚さない一時的な脇道会話（別名 /btw）
```

Grill Me を促すプロンプト文例(Grill Me スキルを導入していなくても、この一文で同種の挙動を引き出せる)。

```text
実装に着手する前に、要件の抜け・曖昧さ・矛盾を洗い出してください。
あなたから推奨回答付きの質問を一問ずつ出し、私は Yes/No 中心で答えます。
前提が固まったと判断できたら、その時点で確定した仕様を箇条書きで要約してください。
```

注記: 承認モードの正典コマンド名は現行の公式リストでは `/permissions`。`/approvals` は過去に存在したが変更・削除された経緯があり、現行版では `/permissions` を使う。`/side` は親スレッドの状況を表示しつつ別トランスクリプトを保つが、本線へ自動マージはされない(脇で得た結論は手動で本線に戻す)という制約がある。

#### 出典

- Codex CLI スラッシュコマンド一覧(`/plan`・`/review`・`/permissions`・`/status`・`/side`・`/btw`・`/fork`): https://developers.openai.com/codex/cli/slash-commands
- Codex 設定リファレンス(`approval_policy`・`sandbox_mode` ほか): https://developers.openai.com/codex/config-reference
- Codex プロンプティング解説(Prompts / Threads / Context / Goal Mode): https://developers.openai.com/codex/prompting
- Goal モードの有効化手順(`features.goals = true` / `codex features enable goals`): https://mehmetbaykar.com/posts/enable-goal-mode-in-codex-cli/
- Goal モード追加の告知(Codex CLI 0.128.0、2026-04-30): https://simonwillison.net/2026/Apr/30/codex-goals/
- Grill Me スキル(一次情報・Matt Pocock): https://github.com/mattpocock/skills
- Grill Me 解説(Matt Pocock 本人ブログ): https://www.aihero.dev/my-grill-me-skill-has-gone-viral

---

## 第7章 画像・スクショ投入（Appshot）— 見せて作らせる

#### これは何？

Codex に文字の指示だけでなく、画像やスクリーンショット（画面を撮った画像。以下スクショ）を一緒に渡せる機能です。筆者はこれを「Appshot」と呼んでいますが、Codex 側の正式な機能名ではなく、実体は「画像入力（image input）」です。つまり「Appshot＝画面を撮って Codex に見せる使い方」の愛称だと思ってください。

やり方は2通りあります。

1つ目は、ターミナル（黒い画面でコマンドを打つツール）から `codex -i 画像ファイル名` という形で画像を添えて起動する方法。`-i` は image（画像）の頭文字です。複数の画像を渡したいときは、カンマ区切りでファイル名を並べます。

2つ目は、TUI（Text User Interface＝ターミナル内で動く対話画面）を起動した状態で、入力欄に画像を貼り付ける方法。クリップボードにコピーした画像を、入力欄で貼り付け操作すると添付されます。

対応している画像形式は PNG / JPEG / GIF / WebP です。スクショやデザイン画像はたいてい PNG か JPEG なので、そのまま渡せます。

#### なぜ"賢くなる"のか

文章だけで画面を説明するのは難しく、伝え漏れが起きます。「上に青いボタン、その下にカードが3枚……」と言葉を尽くしても、余白・色・並びのニュアンスは言葉から抜け落ちます。

画像を渡すと、Codex はレイアウトや配色、要素の位置関係を画像から直接読み取れます。結果として、こちらの説明の手間が減り、出力が「頭の中のイメージ」に近づきます。主な使いどころは2つです。

- デザイン→コード（design-to-code）: UI のデザイン画像やスクショを渡し、「この画面を実装して」と頼む
- エラー画面・グラフの読み取り: 赤いエラー表示やグラフのスクショを渡し、「これは何が起きている？」「原因と直し方は？」と聞く

言葉で往復していた説明を、画像1枚に置き換えられるのが利点です。

#### 使い方（手順）

発想は「言葉で説明しきれないものは、見せて頼む」です。最小の手順はこれだけです。

1. 渡したい画面を用意する（スクショを撮る、またはデザイン画像を用意する）
2. その画像ファイルを、Codex を起動するフォルダに置く（パスが分かれば別の場所でも可）
3. ターミナルで `codex -i ファイル名 "やってほしいこと"` と打つ
4. もう Codex を起動済みなら、入力欄に画像を貼り付けてから指示文を書く

「まず1枚渡して様子を見る」くらいの軽さで試せます。うまく伝わらなければ、画像を足す・指示文を具体的にする、で調整します。

#### 具体例（1つ）

ログイン画面のデザイン画像を実装させるケースです。

Before（言葉だけ）:
> 「中央にメールアドレスとパスワードの入力欄、その下にログインボタンがあるログイン画面を作って」
→ 配置や余白、色がこちらの想定とずれた状態で出てくることがあり、何度も口頭で修正することになりがちです。

After（画像を添える）:
```bash
codex -i login-mockup.png "この画像のログイン画面を React で実装して。入力欄のラベル位置と余白も画像に合わせて"
```
→ Codex が画像から中央寄せのレイアウト、入力欄の縦並び、ボタンの位置と色を読み取り、近い形で実装を出します。説明の往復が減り、初回の出力の精度が上がります。

#### エージェント実装用

ターミナルからの画像入力（CLI）:
```bash
# 画像1枚を添えて起動
codex -i screenshot.png "この画面を実装して"

# 長い形のフラグ名でも同じ
codex --image screenshot.png "この画面を実装して"

# 複数画像はカンマ区切り（スペースを入れない）
codex --image img1.png,img2.jpg "2枚を見比べて差分を説明して"
```

TUI（対話画面）での貼り付け:
```text
1. `codex` と打って対話画面（TUI）を起動する
2. 渡したい画像をクリップボードにコピーしておく
3. 入力欄（コンポーザー）で画像を貼り付ける
   - Linux / Windows: Ctrl+V で貼り付け（公式が明記するのは「対話コンポーザーへの画像ペースト対応」まで。Ctrl+V という具体キーはコミュニティ報告由来で一部未確認）
   - 端末によってはドラッグ&ドロップでも添付できる
4. 続けて指示文を入力して送信する
```

注意（機密の扱い）:
```text
- API キー、トークン、個人情報、社外秘の画面などが写ったスクショは渡さない
- 渡す前に、画面に映り込んだ機密情報をマスク（黒塗り）する
- 画像は外部のモデルに送られて処理される前提で扱う
```

#### 出典

- Codex CLI Features（`codex -i` / `--image`、カンマ区切り複数画像、対話コンポーザーへの画像ペースト）: https://developers.openai.com/codex/cli/features
- TUI での Ctrl+V ペースト挙動・対応形式（PNG/JPEG/GIF/WebP）に関するコミュニティ報告（一部未確認）: https://github.com/openai/codex/issues/2730

---

## 第8章 Claude Code × Codex の両立 — 他社AIを"審査員"に雇う

#### これは何？

ここまでの章は「Codex 単体をどう賢くするか」でした。最終章のテーマは、それと逆向きです。Codex に加えて、もう一つ別の会社のAI（ここでは Anthropic の **Claude Code** — Claude を端末から動かすコーディング用ツール）を同じ作業場に入れ、**役割を分けて両方を使う**。

分け方はシンプルです。

- **実装役（手を動かす人）** … コードを書く・直す。ここでは Claude Code が担当。
- **レビュー役（審査する人）** … 書かれたコードを読んで、抜け・バグ・甘さを指摘する。ここでは Codex が担当。

なぜわざわざ会社を分けるのか。同じモデルに「自分が書いたコードを採点して」と頼むと、自分の判断を肯定しがちな傾向（**sycophancy** = 追従バイアス。相手や自分に都合よく同意してしまう癖）が出やすいからです。「いいと思います」で素通りされると、レビューの意味が薄くなる。実装した本人とは別系統のAIに審査させれば、この「身内の甘さ」を仕組みとして減らせます。野球で、自軍の選手を自軍の監督が判定しない（独立した審判を置く）のと同じ発想です。

#### なぜ"賢くなる"のか

出力の質は「最初に書いた一発」ではなく「指摘 → 修正の往復」で上がります。ここで効くのが「**別系統の目**」です。

- 同じモデル同士のレビューは、得意・不得意のクセまで似ているため、見落としも揃いやすい。
- プロバイダを分けると、片方が見逃した前提や境界条件を、もう片方が拾える確率が上がる。
- さらに後述の「対立レビュー（adversarial review）」を使うと、レビュー役に「この実装判断は本当に正しいのか」と前提から疑わせられる。同意ではなく反証を求める設定です。

つまり「2台のAIを並べる」こと自体が賢さではなく、**実装と審査を別人格に分け、往復させる構造**が出力を締めます。

#### 使い方（手順）

悩み →「Claude に書かせたコード、自分で OK 出すから甘い気がする。別のAIに厳しく見てほしい」。
こう頼むだけ → Claude Code の中から Codex を呼び出すプラグイン **codex-plugin-cc**（OpenAI 公式）を入れ、`/codex:review` と打つ。

最小手順（Claude Code のプロンプト内で実行します）。

1. 前提を用意する。`codex` コマンド（Codex CLI）と Node.js 18.18 以降。Codex に未サインインなら、Claude Code 内で `!codex login`（ChatGPT サブスク or API キー）。
2. プラグインを入れる（次の4行を順に）。
   ```text
   /plugin marketplace add openai/codex-plugin-cc
   /plugin install codex@openai-codex
   /reload-plugins
   /codex:setup
   ```
3. Claude に実装させたあと、`/codex:review` で Codex にレビューさせる。
4. 返ってきた指摘を Claude に渡して直してもらう。これで1往復。

注意（未確認ではなく仕様）: 利用は Codex 側の利用上限にカウントされます。ChatGPT 無料プランでも開始できます。

#### 具体例（1つ）

シナリオ: ログイン処理を Claude Code に実装させた直後。

- **before（Claude 単体）**: Claude「実装しました。問題ないと思います」。本人がそう言うので、そのまま採用。後でメールアドレス未入力時の分岐が抜けていたと判明。
- **after（Codex を審査員に）**:
  1. Claude が実装 → こちらは `/codex:review` と打つ。
  2. Codex の指摘例（イメージ）:「`email` が空文字のケースで例外が `catch` されず 500 を返す。テストも未カバー」。
  3. その指摘文を Claude に渡し「直して」。Claude が分岐とテストを追加。
  4. もう一度 `/codex:review`。残課題なしを確認して採用。

書いた本人ではなく別系統が読むので、「本人が見落とした前提」に当たりやすくなります。前提から疑わせたい時は `/codex:review` の代わりに `/codex:adversarial-review` を使い、`--base main` で「main からの差分だけ」をレビュー対象にもできます。

#### エージェント実装用

以下はコピペで再現できます。出典はすべて OpenAI 公式リポジトリ <https://github.com/openai/codex-plugin-cc> の README。

ステップ1: ルールファイルを symlink で一本化（端末側、リポジトリ直下）。Codex は `AGENTS.md`、Claude Code は `CLAUDE.md` を読むため、片方を実体・もう片方を symlink にして恒久的にズレなくします。

```bash
# AGENTS.md を実体、CLAUDE.md をそのシンボリックリンクにする
ln -sf AGENTS.md CLAUDE.md
# 確認（CLAUDE.md -> AGENTS.md と表示されればよい）
ls -l CLAUDE.md
```

symlink を使えない環境（一部の Windows など）では代替として、`CLAUDE.md` から `@import` で `AGENTS.md` を取り込む、または Codex 側 `~/.codex/config.toml` の `project_doc_fallback_filenames` に `CLAUDE.md` を加えて両者に同じ内容を読ませる方法があります。

```toml
# ~/.codex/config.toml — AGENTS.md が無い時に CLAUDE.md も探す
project_doc_fallback_filenames = ["CLAUDE.md"]
```

ステップ2: codex-plugin-cc を導入（Claude Code のプロンプト内で順に実行）。

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

未サインイン時のログインと、`codex` バイナリ未導入時の手動インストール。

```bash
# Claude Code 内のシェルエスケープ経由でログイン
!codex login
# codex CLI が無ければ（Node.js 18.18 以降が必要）
npm install -g @openai/codex
```

提供されるスラッシュコマンド（README 記載の7個）。

| コマンド | 役割 |
| --- | --- |
| `/codex:review` | 読み取り専用の通常レビュー |
| `/codex:adversarial-review` | 実装判断を challenge する対立レビュー |
| `/codex:rescue` | `codex:codex-rescue` サブエージェントへ作業を委譲 |
| `/codex:status` | バックグラウンドジョブの確認 |
| `/codex:result` | 結果の取得 |
| `/codex:cancel` | ジョブのキャンセル |
| `/codex:setup` | セットアップ／レビューゲート管理 |

主なフラグ:

- `/codex:review` / `/codex:adversarial-review` … `--base <ref>`（ブランチ差分レビュー）, `--wait`, `--background`。`/codex:adversarial-review` はフラグの後ろに注目してほしい点（focus）のテキストを足せる。
- `/codex:rescue` … `--resume`, `--background`, `--model <名>`, `--effort <medium 等>`。例: `/codex:rescue investigate why the tests started failing`。
- `/codex:status` / `/codex:result` / `/codex:cancel` … タスクID を引数に取れる（例: `/codex:status task-abc123`）。

レビューゲート（実装後に自動でレビューを挟む関門）の切り替え:

```text
/codex:setup --enable-review-gate
/codex:setup --disable-review-gate
```

応用（任意）: 逆向きに、Codex を Claude の"部下"として常駐させる構成。Codex を MCP サーバとして Claude Code に登録すると、Claude から Codex をツールとして呼べます。

```bash
# Codex を MCP サーバとして Claude Code に登録
claude mcp add codex -- codex mcp-server
```

並走させる時は、作業ツリーを分けて衝突を避けます。

```bash
# 同じリポジトリを別ディレクトリで並行作業（実装用ブランチを分離）
git worktree add ../proj-review review
```

#### 出典

- OpenAI 公式リポジトリ codex-plugin-cc（導入手順・スラッシュコマンド・フラグ・前提条件の正典）: <https://github.com/openai/codex-plugin-cc>
- 同リポジトリ README（raw、導入4コマンド・コマンド一覧の一次情報）: <https://raw.githubusercontent.com/openai/codex-plugin-cc/main/README.md>
- AGENTS.md のカスタム指示・階層マージ・`project_doc_fallback_filenames`（公式 Codex ドキュメント）: <https://developers.openai.com/codex/guides/agents-md>
- Codex 設定リファレンス（`project_doc_fallback_filenames` ほか）: <https://developers.openai.com/codex/config-reference>
- Codex の MCP 連携（公式）: <https://developers.openai.com/codex/mcp>
- OpenAI Developer Community 告知スレッド「Introducing Codex Plugin for Claude Code」（2026-03-30。投稿者が OpenAI スタッフかは未確認のため補助情報）: <https://community.openai.com/t/introducing-codex-plugin-for-claude-code/1378186>

---

## ゼロから「10倍化」する手順（チェックリスト）

この資料を一気に適用するなら、次の順で進めると土台から固まります（各項目の詳細は対応する章へ）。

- [ ] `/status` で現状（モデル・思考量・承認モード）を確認する（はじめに）
- [ ] `~/.codex/AGENTS.md` とプロジェクト直下の `AGENTS.md` を、コマンド先頭・完了条件・禁止事項の形で整える（第1章）
- [ ] よく使う手順を `~/.codex/skills/<名前>/SKILL.md` に切り出す（第2章）
- [ ] 困りごとに対応する MCP を1つだけ足す（まずは Context7）（第3章）
- [ ] `config.toml` の `model_reasoning_effort` を用途に合わせて設定する（難所は `high`）（第4章）
- [ ] タスクに「完了条件（Done when）」を付け、`/review` を習慣化する（第5章）
- [ ] `/plan` → `/goal` の流れと、承認モードの絞り方を決める（第6章）
- [ ] 画面が絡む依頼は `codex -i` で画像を渡す（第7章）
- [ ] `codex-plugin-cc` を入れ、Claude Code から `/codex:review` で審査させる（第8章）

## 出典（一次情報・集約）

### 【公式】developers.openai.com / github.com/openai
- AGENTS.md（カスタム指示・階層マージ・32KiB上限）: https://developers.openai.com/codex/guides/agents-md
- Configuration Reference（config.toml 全般）: https://developers.openai.com/codex/config-reference
- Config basics: https://developers.openai.com/codex/config-basic
- Best practices: https://developers.openai.com/codex/learn/best-practices
- Prompting（Goal Mode 等）: https://developers.openai.com/codex/prompting
- CLI スラッシュコマンド一覧: https://developers.openai.com/codex/cli/slash-commands
- CLI Features（画像入力 `-i` 等）: https://developers.openai.com/codex/cli/features
- Agent Skills（SKILL.md）: https://developers.openai.com/codex/skills
- カスタムプロンプト非推奨の告知: https://developers.openai.com/codex/custom-prompts
- MCP 連携: https://developers.openai.com/codex/mcp
- Non-interactive mode（`codex exec`）: https://developers.openai.com/codex/noninteractive
- GitHub Action（`openai/codex-action`）: https://developers.openai.com/codex/github-action
- GitHub PR レビュー（`@codex review`）: https://developers.openai.com/codex/use-cases/github-code-reviews
- Memories: https://developers.openai.com/codex/memories
- Quickstart: https://developers.openai.com/codex/quickstart
- codex-plugin-cc（OpenAI公式リポジトリ・Apache-2.0）: https://github.com/openai/codex-plugin-cc
- codex-plugin-cc README（raw・一次情報）: https://raw.githubusercontent.com/openai/codex-plugin-cc/main/README.md
- Context7 MCP（Upstash）: https://github.com/upstash/context7
- Playwright MCP（Microsoft）: https://github.com/microsoft/playwright-mcp
- GitHub 公式 MCP サーバ: https://github.com/github/github-mcp-server

### 【コミュニティ】
- Goal mode 有効化手順: https://mehmetbaykar.com/posts/enable-goal-mode-in-codex-cli/
- Goal mode 追加の告知（Codex CLI 0.128.0 / 2026-04-30）: https://simonwillison.net/2026/Apr/30/codex-goals/
- Grill Me スキル（Matt Pocock）: https://github.com/mattpocock/skills
- Grill Me 解説（aihero.dev）: https://www.aihero.dev/my-grill-me-skill-has-gone-viral
- 画像ペースト挙動（GitHub Issue・一部未確認）: https://github.com/openai/codex/issues/2730
- Codex Plugin 告知スレッド（投稿者の公式性は未確認）: https://community.openai.com/t/introducing-codex-plugin-for-claude-code/1378186

## 未確認・注意事項

断定する前に、以下は「未確認」または環境依存として扱ってください。

- 具体的なモデル名（`gpt-5-codex` 等）・既定モデルは環境/バージョン依存。本文では断定せず、各自 `/status` で確認（第4章）。
- `xhigh`（reasoning effort 最大）は「Responses API のみ／モデル依存」。効かない環境では `high` を使う（第4章・公式注記）。
- SKILL.md（Codexスキル）が Claude Code の Agent Skills と完全互換で「そのまま動く」かは、OpenAI公式の明言を確認できず（未確認）。流用時は手元で動作確認を（第2章）。
- TUI での画像ペーストの具体キー（Ctrl+V）はコミュニティ報告由来で一部未確認。公式が明記するのは「コンポーザーへの画像ペースト対応」まで（第7章）。
- `codex mcp list` 相当の専用コマンドは公式ドキュメントに明記なし（未確認）。接続確認は `/mcp` を使う（第3章）。
- `/goal`（Goal mode）は experimental（実験的）機能で、Codex CLI 0.128.0 以上が必要（第6章）。
- 承認モードのコマンドは現行 `/permissions`。`/approvals` は旧名（第6章）。
- community.openai.com の告知スレッドは投稿者が OpenAI スタッフか未確認。プラグイン仕様の正典は GitHub README（第8章）。

---

*この資料は解説動画の補助・配布用に作成。内容は2026年6月時点の公開情報に基づきます。最新の仕様は各出典の公式ドキュメントで確認してください。*
