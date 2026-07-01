# Claude Code セットアップリファレンス

インストール済みプラグインとモデルの一覧・使用方法まとめ。
`~/.claude/plugins/installed_plugins.json`, `~/.claude/settings.json` などから 2026-07-02 時点の状態を集約。

---

## 1. モデル

### 利用可能なモデル一覧

| モデル名 | Model ID | 特徴 |
|---|---|---|
| Opus 4.8 | `claude-opus-4-8` | 最上位モデル。複雑な設計判断・advisor 用途向け |
| Sonnet 5 | `claude-sonnet-5` | 現在のデフォルトモデル。日常の実装作業向けバランス型 |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | 軽量・高速。サブエージェントでの単純作業（要約、フィルタリング等）向け |
| Fable 5 | `claude-fable-5` | Claude 5 ファミリーの一員 |

### 現在の設定（`~/.claude/settings.json`）

- `model`: `sonnet`（デフォルトモデルは Sonnet 5。`/model` コマンドで変更・保存済み）
- `advisorModel`: `opus`（`advisor` ツールは Opus が担当）
- `effortLevel`: `xhigh`

### モデル切り替え・関連コマンド

- `/model` — セッションのデフォルトモデルを変更（今回 Sonnet 5 に設定）
- `/fast` — Fast mode の切り替え。Opus 4.8 / 4.7 で利用可能。モデルを下げるのではなく、出力を高速化する動作
- サブエージェント起動時に `model` パラメータでモデルを個別指定可能（例: `sonnet`, `opus`, `haiku`, `fable`）

### 使い分けの目安

- **Opus**: アーキテクチャ設計、advisor によるレビュー、難易度の高い判断
- **Sonnet**: 通常の実装・調査タスクの主力モデル
- **Haiku**: `code-review` プラグインのように、PR のスクリーニングや単純作業を並列に大量実行する場面

---

## 2. インストール済みプラグイン

`~/.claude/settings.json` の `enabledPlugins` により、以下 6 件はすべて有効化済み。

### マーケットプレイス

| マーケットプレイス | ソース |
|---|---|
| `claude-plugins-official` | GitHub: `anthropics/claude-plugins-official`（公式） |
| `app-store-review` | GitHub: `safaiyeh/app-store-review-skill`（`autoUpdate: true`） |

### プラグイン一覧

#### superpowers（v5.1.0）

- **概要**: TDD・デバッグ・共同作業パターンなどのコアスキルライブラリ（作者: Jesse Vincent）
- **提供形態**: スキル群（コマンドなし）。`using-superpowers` スキルがセッション開始時に自動ロードされ、他スキルの発見・強制を担う
- **搭載スキル**:
  | スキル名 | 用途 |
  |---|---|
  | `using-superpowers` | 会話開始時に必ず使用。スキルの発見・利用ルールを規定 |
  | `brainstorming` | 機能追加・実装前の要件/設計探索。創造的作業の前に必須 |
  | `writing-plans` | 仕様が固まった複数ステップタスクの実装計画作成 |
  | `executing-plans` | レビューチェックポイント付きで計画を別セッション実行 |
  | `subagent-driven-development` | 独立タスクをサブエージェントに分担させて実装 |
  | `dispatching-parallel-agents` | 共有状態のない独立タスクが2つ以上あるときの並列実行 |
  | `using-git-worktrees` | 作業分離が必要な機能開発の隔離ワークスペース確保 |
  | `test-driven-development` | 実装前にテストを書く（機能追加・バグ修正時） |
  | `systematic-debugging` | バグ・テスト失敗・想定外動作に遭遇した際の体系的調査 |
  | `requesting-code-review` | タスク完了時・マージ前のレビュー依頼 |
  | `receiving-code-review` | レビュー指摘を鵜呑みにせず技術的に検証してから反映 |
  | `verification-before-completion` | 完了・修正・成功を主張する前に検証コマンドを実行 |
  | `finishing-a-development-branch` | 実装完了後のマージ/PR/クリーンアップ選択 |
  | `writing-skills` | 新規スキル作成・既存スキル編集時 |
- **使い方**: 明示的な `/コマンド` ではなく、該当条件に合致すると自動的に `Skill` ツール経由で発火する設計（「1% でも関連しそうなら必ず使う」がルール）

#### feature-dev

- **概要**: コードベース探索・アーキテクチャ設計・品質レビューに特化したエージェント群による機能開発ワークフロー
- **コマンド**: `/feature-dev [機能の説明]`
  - 「コードベース理解 → 曖昧点の質問 → アーキテクチャ設計 → 実装」の順で進行
- **搭載エージェント**（`Agent` ツールから `subagent_type` で指定）:
  - `feature-dev:code-explorer` — 実行パスの追跡、アーキテクチャ層の把握、依存関係のドキュメント化
  - `feature-dev:code-architect` — 既存パターンを踏まえた実装ブループリント（変更ファイル・コンポーネント設計・ビルド手順）の提示
  - `feature-dev:code-reviewer` — バグ・セキュリティ・規約適合性のレビュー（確信度ベースでフィルタリング）

#### code-review

- **概要**: 複数の専門エージェントによる PR 自動コードレビュー（確信度スコアリング付き）
- **コマンド**: `/code-review <PR番号 または 未指定でローカルブランチ>`
  - `/code-review ultra` で GitHub 上のマルチエージェント・クラウドレビューを起動（ユーザー課金対象、Claude 自身は起動不可）
  - `/ultrareview` は同機能の非推奨エイリアス
- **内部フロー**（`/code-review` 実行時）:
  1. Haiku エージェントで PR がレビュー対象かを判定（クローズ/ドラフト/自明/レビュー済みは skip）
  2. Haiku エージェントで関連 CLAUDE.md のパス一覧を取得
  3. Haiku エージェントで PR 概要を要約
  4. Sonnet エージェント 5 体を並列起動し、CLAUDE.md 準拠・バグ・git 履歴・過去 PR コメント・コード内コメント遵守の観点で独立レビュー
  5. 検出した各 issue を Haiku エージェントで確信度スコアリング（0-100）
- **前提**: git リポジトリが必要（未初期化なら `git init` を提案）

#### app-store-review（v1.0.1）

- **概要**: Apple App Store Review Guidelines（2026-06-08 版）に対するコード適合性評価（作者: safaiyeh）
- **対象**: iOS / macOS / tvOS / watchOS / visionOS、Swift / Objective-C / React Native / Expo
- **提供形態**: スキル（`SKILL.md` + `rules/1-safety.md` 〜 `rules/5-legal.md` の詳細ルールファイル）
- **発火条件**: App Store 提出準備、コンプライアンスチェック、審査却下リスクのある実装（決済・ユーザーデータ・センシティブコンテンツ等）を扱う際に自動トリガー

#### swift-lsp（v1.0.0）

- **概要**: Swift 用言語サーバー（SourceKit-LSP）連携。`.swift` ファイルのコードインテリジェンスを提供
- **前提条件**: Xcode または `brew install swift` で `sourcekit-lsp` を PATH に用意する必要あり
- **使い方**: 該当拡張子のファイルを開くと自動的に LSP 経由でコード補完・定義ジャンプ等が有効化される（明示コマンドなし）

#### rust-analyzer-lsp（v1.0.0）

- **概要**: Rust 用言語サーバー（rust-analyzer）連携。`.rs` ファイルのコードインテリジェンスを提供
- **前提条件**: `rustup component add rust-analyzer` または `brew install rust-analyzer` でインストールが必要
- **使い方**: swift-lsp 同様、対象拡張子のファイルを開くと自動有効化

---

## 3. 参考: 設定ファイルの場所

| 内容 | パス |
|---|---|
| 有効化プラグイン一覧 | `~/.claude/settings.json`（`enabledPlugins`） |
| インストール済みプラグインのメタ情報 | `~/.claude/plugins/installed_plugins.json` |
| マーケットプレイス登録情報 | `~/.claude/plugins/known_marketplaces.json` |
| プラグイン本体キャッシュ | `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` |
| グローバル CLAUDE.md | `~/.claude/CLAUDE.md` |
