---
title: 'Claude CodeがAGENTS.mdに対応！v2.1.277の裏側と現場の移行手順'
description: 'Claude Code v2.1.277にて業界標準AGENTS.mdのネイティブ対応が解禁。modsフレームワークによる実装アーキテクチャ、CLAUDE.mdとの優先順位や4つの設定値、旧シンボリックリンク解消手順まで現場目線で深掘り解説します。'
pubDate: '2026-09-19'
categories: ['Claude Code', 'AI Agent', 'AGENTS.md', 'アーキテクチャ']
---

2026年9月19日未明（JST 03:04）、AnthropicのエンジニアであるThariq Shihipar氏（[@trq212](https://x.com/trq212/status/2101009392611278961)）より、**「Claude CodeがAGENTS.mdのネイティブサポートを開始した」**という速報が発表されました。

このアップデートは直後にSimon Willison氏によっても引用・解説され（[Simon Willison's Weblog](https://simonwillison.net/2026/Sep/18/thariq-shihipar/)）、開発者コミュニティで瞬く間に大きな話題となっています。

実はこの対応、コミュニティでは2025年8月に起票された[issue #6235](https://github.com/anthropics/claude-code/issues/6235)（2026年8月にクローズ）や、直近の[issue #31005](https://github.com/anthropics/claude-code/issues/31005)で1年以上にわたり熱狂的な要望が寄せられていたトピックでした。

本記事では、**Claude Code v2.1.277**でリリースされた`AGENTS.md`サポートの仕様、優先順位と4つの設定値、裏側の実装メカニズム（Claude Code mods）、そして今まで現場で行われていた「シンボリックリンク等の旧ワークアラウンドの片付け方」までを徹底的に深掘りします。

---

## 1. 何が変わったのか？基本挙動とフォールバック仕様

これまでClaude Codeは、プロジェクト固有の指示書（コーディング規約、アーキテクチャ、テスト方針など）として、プロジェクトルートの `CLAUDE.md`（または `.claude/CLAUDE.md`）を一貫して読み込む仕様でした。

一方、業界ではOpenAIや各エージェントツール（Cursor、Windsurf、GitHub Copilot、Codexなど）共通のオープンスタンダードとして [AGENTS.md](https://agents.md/) の採用が急速に進んでおり、**「複数ツールで同じ指示書を二重管理したくない」**という強いペインが存在していました。

### フォールバック探索ルール

v2.1.277以降、デフォルトでは以下の優先順位（Fallback機構）で探索されます。

```text
[ Claude Code 起動 ]
         │
         ▼
[ CLAUDE.md は存在するか？ ] ── YES ──▶ [ CLAUDE.md をロードして実行 ]
         │ NO
         ▼
[ AGENTS.md は存在するか？ ] ── YES ──▶ [ AGENTS.md をロードして実行 ]
         │ NO
         ▼
[ プロジェクト固有指示なしで実行 ]
```

1. **`CLAUDE.md` が存在する場合**: 従来通り `CLAUDE.md` が最優先され、`AGENTS.md` は読み込まれません（競合や二重適用を回避）。
2. **`CLAUDE.md` が存在しない場合**: 自動的に `AGENTS.md` をフォールバックとして探索・ロードします。

つまり、**「すでに他ツール用に `AGENTS.md` があるリポジトリ」では、何の設定も追加せずにClaude Codeを立ち上げるだけで自動認識される**ようになりました。

---

## 2. 4つの設定値（Project Instructions モード）

公式ドキュメント（[code.claude.com/docs/en/memory#agents-md](https://code.claude.com/docs/en/memory#agents-md)）によると、指示書の読み込み挙動は設定で明示的にコントロール可能です。

ターミナルで `/config` を実行するか、設定ファイルで以下の4つのモードを切り替えられます。

| 設定値（Mode） | 挙動の詳細 | 推奨ユースケース |
| :--- | :--- | :--- |
| **`fallback`**（デフォルト） | `CLAUDE.md` を優先。なければ `AGENTS.md` をロード | 通常の開発リポジトリ（最も安全で後方互換性が高い） |
| **`claude-only`** | `CLAUDE.md` のみを探索。`AGENTS.md` は完全無視 | Claude特化の高度マクロを厳格に適用したい専任環境 |
| **`agents-only`** | `AGENTS.md` のみを探索。`CLAUDE.md` を無視 | CursorやCopilotと完全統合し、規約を1ファイルに統一したい現場 |
| **`disabled`** | プロジェクト指示書の自動読み込みを完全無効化 | 軽量なワンショット実行や、指示書によるコンテキスト圧迫を避けたい場合 |

チーム開発において「リポジトリ内にはCursor用の `AGENTS.md` を置いているが、Claude Code独自の実験用指示書をローカルでのみ使いたい」といった場合でも、スコープに合わせて柔軟にハンドリングできます。

---

## 3. アーキテクチャ深掘り：なぜ「Claude Code mods」で実装されたのか？

今回のアップデートで技術的に最も興味深いのは、Anthropicがこの機能を**コアバイナリのハードコードではなく、「Claude Code mods」という拡張機構の上に実装した点**です。

ソースコードは公開リポジトリの [`mods/agents-md`](https://github.com/anthropics/claude-code/tree/main/mods/agents-md) に配置されています。

### Claude Code mods の思想

Simon Willison氏も指摘している通り、Claude Codeは現在、ハーネス（Harness）全体のモジュール化・プラガブル化を進めています。

```text
[ Claude Code Core Runtime ]
           │
     ┌─────┴─────────────────────────┐
     ▼                               ▼
[ Core Features ]             [ Claude Mods ]
- Agent Loop                  - mods/agents-md (今回追加)
- Tool Execution              - Custom Instruction Loaders
- Context Management          - Local Hook Injectors
```

`mods/agents-md` は、セッションライフサイクルの初期化フックをリッスンし、以下のような処理を行っています：

1. **ファイルシステムプローブ**: 現在のワーキングディレクトリおよびGitルートにおける `CLAUDE.md` と `AGENTS.md` の存在確認。
2. **コンフィグ照合**: ユーザーが設定したモード（`fallback`, `agents-only` 等）の評価。
3. **プロンプトコンテキストへの安全な注入**: システムプロンプトのインストラクション領域へ、標準化されたセクション見出しとともにMarkdown本文を注入。

#### なぜこのアーキテクチャが開発者にとって重要なのか？

Anthropicが指示書の読み込みを単一のモジュール（mod）として切り出したことは、**「将来的には開発者が独自のカスタムmodを作成し、自社専用のルールローダー（例: `SECURITY_POLICY.md` や社内コンプライアンスチェッカー）をプラグインとして差し込めるようになる未来」**を強く示唆しています。

---

## 4. 現場のハマりどころと「使えない条件」

一見すると「移行して終わり」に見える機能ですが、本番・実務環境ではいくつか注意すべきボトルネックがあります。

### ① 機能フラグを取得しないセッション（オフライン・CI環境）

Claude Codeの一部の新機能やmodは、起動時にAnthropicの機能フラグ（Feature Flags）サービスを参照して動的に有効化されます（[code.claude.com/docs/en/env-vars](https://code.claude.com/docs/en/env-vars) 参照）。

以下のような環境では、`AGENTS.md` の読み込みが発動しないケースがあります：

- **オフライン開発環境**: 外部ネットワークへの接続が制限された閉域網
- **CI/CDパイプライン**: ジョブ実行時に外部通信をブロックしている環境
- **明示的なmod無効化**: `CLAUDE_DISABLE_MODS=1` などの環境変数がセットされている場合

> **トラブルシューティング:**
> Claude Codeを立ち上げた際、初期ログ（ロードされたコンテキスト）に `Loaded project instructions from AGENTS.md` の表示が出ているかを必ず確認してください。表示がない場合は機能フラグが降ってきていないか、設定が `claude-only` になっている可能性があります。

### ② CLAUDE.md 固有ディレクティブの非互換

`CLAUDE.md` には、Claude Code特有のメモリショートカットや自動更新用メタタグ、固有のセクション構文が記述されていることがあります。

`AGENTS.md` にそのまま内容を丸写しした場合、他ツール（Cursorなど）ではそれらのClaude専用構文が不要なノイズとなり、逆にClaude Code側でも汎用Markdownとしてフラットに扱われるため、意図したメモリ更新フックが機能しない場合があります。

---

## 5. 旧ワークアラウンドの片付けガイド（移行ステップ）

これまでエンジニアは、Claude Codeと他エージェントを共存させるために以下のどちらかのワークアラウンドを行っていました。これらは**今すぐクリーンアップすることを強く推奨**します。

### パターンA：シンボリックリンク運用の解消

多くの現場で `ln -s AGENTS.md CLAUDE.md` というシンボリックリンクが作成されていました。

```bash
# 旧運用の負債
$ ls -l
lrwxrwxrwx 1 user user  9 Sep 01 10:00 CLAUDE.md -> AGENTS.md
-rw-r--r-- 1 user user 1.2K Sep 01 10:00 AGENTS.md
```

#### なぜ削除すべきか？
- Windows環境（Git for Windows / WSL）とmacOS/Linux間でシンボリックリンクのコミット破損が発生しやすい。
- Claude Codeが将来的に両ファイルの差分マージや特定モードを導入した際、リンク循環や解決エラーの原因になる。

**クリーンアップ手順:**
```bash
# 1. シンボリックリンクを削除
rm CLAUDE.md

# 2. Gitの状態を確認してコミット
git add CLAUDE.md
git commit -m "chore: remove CLAUDE.md symlink in favor of native AGENTS.md support"
```

### パターンB：CLAUDE.md からの `@AGENTS.md` 参照の解消

`CLAUDE.md` の中に `@AGENTS.md` と1行だけ書いて読み込ませていた運用です。

```markdown
<!-- 旧 CLAUDE.md -->
@AGENTS.md
```

#### なぜ削除すべきか？
`CLAUDE.md` が残っている限り、Claude Codeは「`CLAUDE.md` が存在する」と判定し、新設された `AGENTS.md` の直接最適化ローダーをバイパスしてインクルード処理を行うため、不要なコンテキスト層が生成されます。

- **マルチエージェントに完全統一する場合**: `CLAUDE.md` を削除し、`AGENTS.md` のみに一本化。
- **Claude特化の指示を併用したい場合**: `AGENTS.md` には共通規約（アーキテクチャ、Lint、ビルド手順）を残し、`CLAUDE.md` にはClaude Code固有のコマンドエイリアスや自動テスト指示のみを記載する。

---

## 6. まとめ：1リポジトリ・複数エージェントの時代へ

OpenAI主導で立ち上がり、中立組織へと寄贈された `AGENTS.md`。そこにClaude Codeが公式対応したことで、**「AIエージェントごとに指示書ファイルが乱立する時代」は決定的な終焉**を迎えました。

```text
               ┌── Cursor
               ├── Windsurf
[ AGENTS.md ] ─┼── GitHub Copilot
 (Single Source)├── Codex
               └── Claude Code (v2.1.277〜) ★NEW
```

1つの `AGENTS.md` をリポジトリのルートに置き、チーム全員が好みのAIエージェントを自由に選択して同じコンテキストで開発する――その理想的な開発環境が、今回のアップデートによって完全な実用フェーズに入りました。

まずは `npm install -g @anthropic-ai/claude-code@latest` でバージョンを `2.1.277` 以上に更新し、長年放置していたシンボリックリンクを削除することから始めてみてください。
