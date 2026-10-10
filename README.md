# Claude Code セットアップリファレンス

インストール済みプラグインとモデルの一覧・使用方法まとめ。
`~/.claude/plugins/installed_plugins.json`, `~/.claude/settings.json` などから 2026-10-11 時点の状態を集約。

---

## 1. モデル

### 利用可能なモデル一覧（Claude 5 ファミリー）

| モデル名 | Model ID | 特徴 |
|---|---|---|
| Opus 5.5 | `claude-opus-5-5` | 最上位モデル。現在のデフォルト。設計判断・実装・advisor 用途 |
| Sonnet 5.5 | `claude-sonnet-5-5` | バランス型。本環境では機械的な探索・検索のみに使用（後述のポリシー） |
| Haiku 5.5 | `claude-haiku-5-5` | 軽量・高速。本環境では使用禁止（後述のポリシー） |
| Fable 5.1 | `claude-fable-5-1` | Claude 5 ファミリーの一員 |

### 現在の設定（`~/.claude/settings.json`）

- `model`: `opus`（デフォルトモデルは Opus 5.5）
- `advisorModel`: `opus`（`advisor` ツールは Opus が担当）
- `effortLevel`: `xhigh`（全体の既定）
- `modelSettings.claude-opus-5-5.effortLevel`: `medium`（Opus 5.5 使用時はモデル別設定で上書き）
- `permissions.defaultMode`: `auto`

### モデル切り替え・関連コマンド

- `/model` — セッションのデフォルトモデルを変更
- `/fast` — Fast mode の切り替え。Opus のまま出力を高速化する（モデルを下げるわけではない）
- サブエージェント起動時に `model` パラメータでモデルを個別指定可能（`opus`, `sonnet`, `haiku`, `fable`）

### 使い分けポリシー（`~/.claude/CLAUDE.md` の Model Policy）

- **Opus**: 設計・計画・実装・レビュー・デバッグなど思考を伴うタスクはすべて Opus
- **Sonnet**: 機械的な探索・検索・grep 的走査のみ可
- **Haiku**: 全タスクで使用禁止
- subagent 起動時は `model` を必ず明示指定する。`feature-dev` 系 agent は frontmatter で `model: sonnet` 固定のため、思考タスクでは `model: "opus"` で上書きする

---

## 2. インストール済みプラグイン

`~/.claude/settings.json` の `enabledPlugins` により、以下 7 件はすべて有効化済み（user scope と `/Users/YutoFujioka` の project scope の両方に登録）。

### マーケットプレイス

| マーケットプレイス | ソース |
|---|---|
| `claude-plugins-official` | GitHub: `anthropics/claude-plugins-official`（公式） |
| `app-store-review` | GitHub: `safaiyeh/app-store-review-skill`（`autoUpdate: true`） |

### バージョン一覧

| プラグイン | バージョン | 最終更新 |
|---|---|---|
| `superpowers` | 6.4.1 | 2026-09-24 |
| `feature-dev` | `b8e53f1c05df`（commit SHA） | 2026-10-10 |
| `code-review` | `b8e53f1c05df`（commit SHA） | 2026-10-10 |
| `frontend-design` | `b8e53f1c05df`（commit SHA） | 2026-10-10 |
| `app-store-review` | 1.3.3 | 2026-10-10 |
| `swift-lsp` | 1.0.0 | 2026-07-09 |
| `rust-analyzer-lsp` | 1.0.0 | 2026-07-09 |

### プラグイン詳細

#### superpowers（v6.4.1）

- **概要**: TDD・デバッグ・共同作業パターンなどのコアスキルライブラリ（作者: Jesse Vincent）
- **提供形態**: スキル群（コマンドなし）。`using-superpowers` スキルが SessionStart hook でセッション開始時に自動ロードされ、他スキルの発見・強制を担う
- **搭載スキル**:
  | スキル名 | 用途 |
  |---|---|
  | `using-superpowers` | 会話開始時に必ず使用。スキルの発見・利用ルールを規定 |
  | `brainstorming` | 機能追加・実装前の要件/設計探索。創造的作業の前に必須 |
  | `writing-plans` | 仕様が固まった複数ステップタスクの実装計画作成 |
  | `executing-plans` | 計画を現セッションで自分自身が実装者として実行（インライン実行） |
  | `subagent-driven-development` | 独立タスクをサブエージェントに分担させて実装 |
  | `dispatching-parallel-agents` | 共有状態のない独立タスクが2つ以上あるときの並列実行 |
  | `using-git-worktrees` | 作業分離が必要な機能開発の隔離ワークスペース確保（native ツール or git worktree） |
  | `test-driven-development` | 実装前にテストを書く（機能追加・バグ修正時） |
  | `systematic-debugging` | バグ・テスト失敗・想定外動作に遭遇した際の体系的調査 |
  | `requesting-code-review` | タスク完了時・マージ前のレビュー依頼 |
  | `receiving-code-review` | レビュー指摘を鵜呑みにせず技術的に検証してから反映 |
  | `verification-before-completion` | 完了・修正・成功を主張する前に検証コマンドを実行 |
  | `finishing-a-development-branch` | 実装完了後のマージ/PR/クリーンアップ選択 |
  | `writing-skills` | 新規スキル作成・既存スキル編集時 |
  | `diagnosing-superpowers` | **v6 で追加**。セッションがうまくいかなかった理由（繰り返し作業・計画無視・スキル不発火・遅い/高い等）の診断や、メンテナ向けバグレポート作成 |
- **使い方**: 明示的な `/コマンド` ではなく、該当条件に合致すると自動的に `Skill` ツール経由で発火する設計（「1% でも関連しそうなら必ず使う」がルール）

#### feature-dev

- **概要**: コードベース探索・アーキテクチャ設計・品質レビューに特化したエージェント群による機能開発ワークフロー
- **コマンド**: `/feature-dev [機能の説明]`
  - 「コードベース理解 → 曖昧点の質問 → アーキテクチャ設計 → 実装」の順で進行
- **搭載エージェント**（`Agent` ツールから `subagent_type` で指定。いずれも frontmatter 既定は `model: sonnet`）:
  - `feature-dev:code-explorer` — 実行パスの追跡、アーキテクチャ層の把握、依存関係のドキュメント化
  - `feature-dev:code-architect` — 既存パターンを踏まえた実装ブループリント（変更ファイル・コンポーネント設計・ビルド手順）の提示
  - `feature-dev:code-reviewer` — バグ・セキュリティ・規約適合性のレビュー（確信度ベースでフィルタリング）

#### code-review（プラグイン: `/code-review:code-review`）

- **概要**: 複数の専門エージェントによる GitHub PR 自動コードレビュー（確信度スコアリング付き）。結果は `gh pr comment` で PR にコメントする
- **コマンド**: `/code-review:code-review <PR番号>`
  - 組み込みの `/code-review`（後述）と名前が重なるため、プラグイン版は名前空間付きで呼ぶ
- **内部フロー**:
  1. Haiku エージェントで PR がレビュー対象かを判定（クローズ/ドラフト/自明/レビュー済みは skip）
  2. Haiku エージェントで関連 CLAUDE.md のパス一覧を取得
  3. Haiku エージェントで PR 概要を要約
  4. Sonnet エージェント 5 体を並列起動し、CLAUDE.md 準拠・バグ・git 履歴・過去 PR コメント・コード内コメント遵守の観点で独立レビュー
  5. 検出した各 issue を Haiku エージェントで確信度スコアリング（0-100）
  6. スコア 80 未満を除外し、再度対象判定してから PR にコメント
- **注意**: 内部で Haiku を使う設計のため、CLAUDE.md の Haiku 禁止ポリシーとは衝突する。基本は組み込みの `/code-review` を使う

#### frontend-design

- **概要**: 汎用的な "AI 的" デザインを避け、独自性のあるプロダクション品質のフロントエンド UI/UX を生成するスキル（作者: Anthropic）
- **提供形態**: スキル（`SKILL.md` のみ、コマンドなし）。Web コンポーネント・ページ・アプリの構築依頼時に自動発火
- **発火条件**: 新規 UI の構築や既存 UI の作り直しを含むタスク
- **特徴**:
  - 実装前に明確な美的方向性（ミニマル / マキシマル / レトロフューチャー / エディトリアル等）を選択
  - タイポグラフィ・配色・アニメーション・細部まで作り込んだ実働コード（HTML/CSS/JS, React, Vue 等）を生成
  - 意図性を重視（強度ではなく一貫したコンセプトの徹底）

#### app-store-review（v1.3.3）

- **概要**: Apple App Store Review Guidelines（2026-06-08 版、2026-08-29 時点で最新と確認済み）に対するコード適合性評価（作者: safaiyeh）
  - 6 月以降の発表（SNS 関連の年齢レーティング質問の 2026-09 義務化、韓国の年齢レーティング変更、ブラジル/EU の代替決済・配信条件）も反映
- **対象**: iOS / macOS / tvOS / watchOS / visionOS、Swift / Objective-C / React Native / Expo
- **提供形態**: スキル（`SKILL.md` + `rules/1-safety.md` 〜 `rules/5-legal.md` の詳細ルールファイル）
- **発火条件**: App Store 提出準備、コンプライアンスチェック、審査却下リスクのある実装（決済・ユーザーデータ・センシティブコンテンツ等）を扱う際に自動トリガー
- **v1.3 系の変更点**: API 呼び出しや SDK import だけで却下と断定せず、設定・データフローの証拠を確認する方針に。スキルの誤り・古い記述をメンテナへ報告するフィードバック機能を追加

#### swift-lsp（v1.0.0）

- **概要**: Swift 用言語サーバー（SourceKit-LSP）連携。`.swift` ファイルのコードインテリジェンスを提供
- **前提条件**: Xcode または `brew install swift` で `sourcekit-lsp` を PATH に用意する必要あり
- **使い方**: 該当拡張子のファイルを開くと自動的に LSP 経由でコード補完・定義ジャンプ等が有効化される（明示コマンドなし）

#### rust-analyzer-lsp（v1.0.0）

- **概要**: Rust 用言語サーバー（rust-analyzer）連携。`.rs` ファイルのコードインテリジェンスを提供
- **前提条件**: `rustup component add rust-analyzer` または `brew install rust-analyzer` でインストールが必要
- **使い方**: swift-lsp 同様、対象拡張子のファイルを開くと自動有効化

---

## 3. 組み込みコマンド・スキル（プラグイン外）

Claude Code 本体に同梱されている主なもの。

| コマンド | 用途 |
|---|---|
| `/code-review [low\|medium\|high\|xhigh\|max\|ultra] [対象]` | 現在の diff / PR 番号 / ブランチ / パスをレビュー。effort レベルで検出数と確度を調整。`--comment` で PR にインラインコメント、`--fix` で指摘を作業ツリーに適用、`--max-findings <n\|all>` で件数指定 |
| `/code-review ultra [PR#]` | クラウド上のマルチエージェント深掘りレビュー（ユーザー課金、Claude 自身は起動不可）。`/ultrareview` は非推奨エイリアス |
| `/simplify` | 変更コードの再利用・簡素化・効率化の観点で整理して修正（バグ探しはしない） |
| `/security-review` | 現ブランチの未マージ変更のセキュリティレビュー |
| `/init` | プロジェクトの CLAUDE.md を生成 |
| `/loop [間隔] <prompt>` | プロンプトを一定間隔で繰り返し実行（間隔省略でモデルが自己ペース） |
| `/schedule` | cron でクラウドエージェント（routine）を定期/単発実行 |
| `/run` | プロジェクトのアプリを起動して動作確認 |
| `/fewer-permission-prompts` | 過去のトランスクリプトから読み取り系コマンドの allowlist を提案 |
| `/artifacts` | 作成した Artifact の一覧（`o` で開く / `c` でリンクコピー） |

このほか claude.ai から同期されたスキル（`~/.claude/skills/synced/`）: `docs`, `docx`, `pdf`, `pptx`, `xlsx`, `google-workspace`, `deep-research`, `skill-creator`, `import-memory`, `morning`, `chrome-browser`, `built-in-browser`, `computer-use`。

---

## 4. ユーザー定義のカスタマイズ

### コマンド・スキル

| 名前 | 場所 | 用途 |
|---|---|---|
| `/handoff` | `~/.claude/commands/handoff.md` | 現在のタスク進捗を `<repo root>/.claude/handoff.md` に上書きで書き出し、`/clear` できる状態にする |
| `long-horizon-memory` | `~/.claude/skills/long-horizon-memory` | `~/Desktop/dotfile-main/claude/skills/long-horizon-memory` へのシンボリックリンク。**現在リンク先が存在せず無効** |

### Hooks / その他の設定

- **SessionStart hook**: `<repo root>/.claude/handoff.md` があれば、最終更新日時つきで context に注入する（session 開始・`/clear`・compact・resume 時）
- **statusLine**: `~/.claude/statusline-command.sh` を実行
- **env**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`（Agent Teams 有効化）、Android SDK/NDK パス
- **permissions**: `git *` / `gh *` は allow、`gh pr create*` のみ ask
- その他: `autoCompactEnabled`, `autoContinueAtUsageLimit`, `remoteControlAtStartup`, 入力待ち/エージェント完了の push 通知を有効化

### 運用ルール（`~/.claude/CLAUDE.md` 抜粋）

- plan → auto の 2 段構成。`feat` は必ず plan を経由
- main / master への直接コミット禁止。`feat/` `fix/` `docs/` 等のブランチを切る
- タスクの区切りで handoff を更新して `/clear`

---

## 5. 参考: 設定ファイルの場所

| 内容 | パス |
|---|---|
| 有効化プラグイン・hooks・権限など | `~/.claude/settings.json` |
| インストール済みプラグインのメタ情報 | `~/.claude/plugins/installed_plugins.json` |
| マーケットプレイス登録情報 | `~/.claude/plugins/known_marketplaces.json` |
| プラグイン本体キャッシュ | `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` |
| ユーザー定義コマンド | `~/.claude/commands/` |
| ユーザー定義・同期スキル | `~/.claude/skills/` |
| ステータスライン | `~/.claude/statusline-command.sh` |
| グローバル CLAUDE.md | `~/.claude/CLAUDE.md` |
