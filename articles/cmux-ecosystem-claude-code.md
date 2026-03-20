---
title: "cmuxで変わるClaude Codeのマルチプロジェクト開発体験"
emoji: "🖥"
type: "tech"
topics: ["claudecode", "cmux", "ai", "terminal", "agent"]
published: false
slug: "cmux-ecosystem-claude-code"
---

## はじめに

Claude Code、とても便利ですよね。でも使い込んでいくと、こんな不満が出てきませんか？

- **サブエージェントが何をやっているか見えない**（Agent ツールはブラックボックス）
- **複数プロジェクトをまたぐ作業がつらい**（Claude Code はカレントディレクトリドリブンなので）
- **別リポジトリでの作業を AI に任せたいけど、cd して別セッション立ち上げて…が面倒**

これらの課題を解決するために、[cmux](https://cmux.dev) というターミナルマルチプレクサを軸にしたエコシステムを作りました。この記事では、そのエコシステムを構成する4つのリポジトリを紹介します。

## cmux とは（30秒で）

![cmux で4つのリポジトリを同時に開いている様子](/images/cmux-ecosystem-demo.png)
*cmux 上で using-cmux / cmux-team / cfork / cmux-remote の4リポジトリを同時に管理*

[cmux](https://cmux.dev) は AI エージェント向けに設計されたターミナルマルチプレクサです。tmux のようにペインを分割できるのに加えて、**CLI から操作を完全に自動化できる**のが最大の特徴です。

```bash
# ペイン分割
cmux new-split right

# 別ペインにコマンドを送信（surface:2 = surface番号で指定）
cmux send --surface surface:2 "echo hello\n"

# 別ペインの画面を読み取り
cmux read-screen --surface surface:2
```

各ペインには `surface:1`, `surface:2` のような**短縮 refs**（連番）が振られ、この番号だけで操作先を指定できます。UUID を覚える必要はありません。AI にとっても人間にとっても直感的に操作できる仕組みです。

つまり、AI がターミナルを「見て」「操作できる」。これが全ての起点です。

## なぜ Ghostty や素の Claude Code では足りないのか

Ghostty や iTerm2 は素晴らしいターミナルですが、**AI が他のペインを操作する手段がありません**。Claude Code の組み込み Agent ツールも便利ですが、こんな問題があります。

| 課題 | 素の Claude Code | cmux 経由 |
|------|-----------------|----------|
| サブエージェントの出力 | 見えない | リアルタイムで見える |
| 途中介入 | できない | いつでも可能 |
| 別プロジェクトへの操作 | cd + 新セッションが必要 | `cmux new-workspace` で即座に |
| デバッグ | 結果だけ返ってくる | I/O全体を確認できる |

**「AI がやっていることが見える」だけで、QoL が劇的に上がります。**

## 1. using-cmux — AI が cmux を操作するためのスキル

**リポジトリ**: [hummer98/using-cmux](https://github.com/hummer98/using-cmux)

### 何ができるのか

Claude Code に「cmux の操作方法」を教えるスキルパッケージです。インストールすると、Claude Code が以下のことを自律的にできるようになります。

- ペインの分割・管理
- 別ペインへのコマンド送信と画面読み取り
- **サブエージェントの起動 → 監視 → 結果回収**（これが本丸）
- ワークスペースをまたいだ操作
- ステータス表示や通知

### なぜ作ったのか

既存の cmux スキル（[hashangit/cmux-skill](https://github.com/hashangit/cmux-skill)）はブラウザ自動化の記述が全体の約50%を占めていて、最も重要な**サブエージェント操作パターンが埋もれていました**。

using-cmux ではその構成を見直して、サブエージェント操作を中核に据え直しています。

### サブエージェントの起動パターン

ここが一番伝えたいところです。Claude Code が別ペインに Claude Code を立ち上げて、タスクを投げて、結果を回収する——という一連の流れがパターン化されています。

```bash
# 1. ワークスペース作成
WS=$(cmux new-workspace "task-name")

# 2. Claude Code を起動
cmux send --workspace "$WS" --surface surface:1 \
  "claude --dangerously-skip-permissions\n"

# 3. Trust 確認を待つ（プロンプトの出現を検出）

# 4. プロンプトを送信
cmux send --workspace "$WS" --surface surface:1 "${PROMPT}"
cmux send-key --workspace "$WS" --surface surface:1 return

# 5. 完了を検出して結果を回収
cmux read-screen --workspace "$WS" --surface surface:1 --scrollback 500
```

ポイントは、**全ての操作が CLI 経由なので、AI が完全に自動化できる**ということです。人間は横で見ているだけで OK です（もちろん介入もできます）。

### インストール

```bash
# Plugin（推奨 — スキル + コマンド + フック全対応）
/plugin marketplace add hummer98/using-cmux
/plugin install using-cmux

# Agent Skills（スキルのみ）
npx skills add hummer98/using-cmux
```

cmux セッション内で Claude Code を起動すると、環境変数 `CMUX_SOCKET_PATH` を検出して自動的にスキルがロードされます。設定不要です。

### タブに surface 番号を自動表示

Plugin としてインストールすると、**SessionStart フック**により Claude Code 起動時にタブタイトルが自動的に `[87] Claude Code` のような形式に変わります。

これにより、`cmux tree` で表示される surface 番号とタブタイトルが一致し、「どのペインが surface:87 なのか」が一目でわかります。複数のサブエージェントを同時に走らせているとき、操作対象を間違えるリスクが大幅に減ります。

```bash
# タブタイトルから surface 番号がすぐわかる
#   [87] Claude Code  ← surface:87
#   [91] Claude Code  ← surface:91
#   [108] Claude Code ← surface:108
```

> **Note**: この機能は Plugin インストール時のみ有効です。Agent Skills（`npx skills add`）ではフックが配布されないため、タブタイトルの自動変更は行われません。

---

## 2. cmux-team — サブエージェントをチームとして動かす

**リポジトリ**: [hummer98/cmux-team](https://github.com/hummer98/cmux-team)

### using-cmux の次のステップ

using-cmux で「1体のサブエージェントを操作する」パターンが確立できました。では次は？ **複数のサブエージェントを並列で動かしたい**ですよね。

cmux-team は、Claude Code をオーケストレータ（Conductor）として、複数のサブエージェントを同時に起動・管理・同期するフレームワークです。

### アーキテクチャ

```
Conductor（親 Claude Code）
  ├── Researcher × 3  ← 並列リサーチ
  ├── Architect × 1   ← 設計
  ├── Implementer × 2 ← 並列実装
  ├── Reviewer × 1    ← レビュー
  └── Tester × 1      ← テスト
```

各エージェントは**独立した cmux ワークスペース**で動きます。Conductor は `cmux read-screen` で各エージェントの進捗をリアルタイムに監視し、`cmux wait-for` で完了を待ちます。

### スラッシュコマンドで制御

開発フェーズに対応した11個のスラッシュコマンドが用意されています。

```
/team-init       → チーム初期化
/team-research   → 並列リサーチ（3エージェント）
/team-design     → 設計 + レビュー
/team-impl       → 並列実装
/team-review     → コードレビュー
/team-test       → テスト実行
/team-status     → 進捗確認
/team-disband    → 全エージェント終了
```

### エージェントロールのテンプレート

各ロール（Researcher、Architect、Implementer など）には専用のプロンプトテンプレートがあります。Conductor が `{{PROJECT_NAME}}` や `{{TASK_DESCRIPTION}}` などの変数を埋めてサブエージェントに送信します。

### ワークスペース分離の重要性

開発中に学んだ教訓があります。**Conductor とサブエージェントを同じワークスペースに置くと、ペインが狭くなりすぎて `cmux send` や `cmux read-screen` が壊れます。**

解決策はシンプルで、Conductor を workspace:1 に置き、サブエージェントは workspace:2 以降に分散させます。

```
workspace:1  → Conductor（操作する側）
workspace:2  → Researcher × 3（3分割）
workspace:3  → Implementer × 2 + Reviewer
workspace:4  → Tester + DocKeeper
```

### インストール

```bash
# Plugin（推奨）
claude /plugin install hummer98/cmux-team

# Agent Skills
npx skills add hummer98/cmux-team
```

---

## 3. cfork — 会話を一瞬でフォークする

**リポジトリ**: [hummer98/cfork](https://github.com/hummer98/cfork)

### 「ちょっと別のアプローチも試したい」

Claude Code で作業していると、「今の会話コンテキストを保ったまま別のアプローチを試したい」ことがあります。でも普通にやると、新しいターミナルを開いて `claude --continue --fork-session` を打って…と面倒です。

cfork は、これを一瞬に圧縮します。

### `!cfork` — シェルから直接実行（推奨）

本丸はこちらです。Claude Code の `!` プレフィックス（シェルエスケープ）を使って直接実行します。

```
!cfork        → 右に新しいペインが開いて、会話がフォークされる
!cfork down   → 下方向にフォーク
```

LLM を経由しないので**一瞬で完了**します。`/cfork`（スラッシュコマンド）だと LLM がプロンプトを解釈してから実行するため数秒かかりますが、`!cfork` はシェルスクリプトを直接叩くだけなのでほぼゼロ遅延です。

実装はびっくりするほどシンプルです。

```bash
S=$(cmux new-split "${DIRECTION:-right}")
cmux send --surface "$S" "claude --continue --fork-session\n"
```

たった2行。でもこの2行が、**「気軽にブランチを切る」感覚で会話をフォークできる**体験を生みます。Git のブランチのように、会話を分岐させて並行で検証できるようになります。

---

## 4. cmux-remote — iPhone からターミナルを監視する

**リポジトリ**: [hummer98/cmux-remote](https://github.com/hummer98/cmux-remote)

### AI が働いている間、席を離れたい

cmux-team で複数エージェントを走らせると、完了まで数分〜数十分かかることがあります。その間ずっと PC の前に座っている必要はないはずです。

cmux-remote は、**iPhone から cmux のワークスペースをリアルタイムに監視できる PWA**（Progressive Web App）です。

### 仕組み

```
iPhone (PWA: React + xterm.js)
    ↕ WebSocket
Bridge Server (Bun + Hono)
    ↕ Unix Domain Socket
cmux (~/.../cmux.sock)
```

Bridge サーバーが cmux のソケットと WebSocket の間を透過的に中継します。iPhone 側では xterm.js でターミナル出力をそのまま表示します。

### ジェスチャーで操作

iPhone ならではの操作体系を備えています。

| ジェスチャー | 動作 |
|-------------|------|
| 2本指で上下スワイプ | ワークスペース切り替え |
| 2本指で左右スワイプ | ペイン切り替え |
| ハンバーガーメニュー | ワークスペース一覧 |

cmux-team で複数ワークスペースにエージェントを分散させている場合、スワイプだけで各エージェントの進捗を順番にチェックできます。

### セットアップ

```bash
# サーバー起動（ポート 3456）
cd server && bun run src/index.ts

# Tailscale や Cloudflare Tunnel 経由で iPhone からアクセス
```

認証はネットワーク層（Tailscale の P2P 接続など）に委ねる設計です。ホーム画面に追加すればネイティブアプリのように使えます。

---

## おまけ: plugin-packager — スキルの配布をもっと楽に

**リポジトリ**: [hummer98/plugin-packager](https://github.com/hummer98/plugin-packager)

### スキル配布のつらさ

Claude Code のスキルには、現状3つの配布方法があります。

1. **Plugin** — `/plugin install` で導入。スキル + コマンド + フックすべて対応
2. **Agent Skills** — `npx skills add` で導入。スキルのみ
3. **手動** — `install.sh` で `~/.claude/` にコピー

問題は、**どの方法に対応するかをリポジトリごとに手動で整備しないといけない**ことです。`plugin.json` の作成、ディレクトリ構造の整理、README のインストールセクション…毎回同じ作業をやることになります。

### plugin-packager がやること

`/package` コマンドを実行すると、リポジトリを走査して以下を自動化します。

1. スキル・コマンド・フックの検出
2. 最適な配布方法の判断
3. Plugin 構造（`.claude-plugin/`）への変換
4. マニフェスト（`plugin.json`, `marketplace.json`）の生成
5. README のインストールセクション生成

### 配布方法の自動判断

| 検出されたコンポーネント | 推奨配布方法 |
|------------------------|-------------|
| スキルのみ | Agent Skills を主体に |
| スキル + コマンド | Plugin を推奨 |
| スキル + コマンド + フック | Plugin 必須 |

### 本音を言うと

正直なところ、**package.json のような依存解決とインストールリストが欲しい**です。

今は各スキルが独立していて、「using-cmux を入れたら cmux-team も一緒に入る」みたいなことができません。npm の `dependencies` のような仕組みが Claude Code のスキルエコシステムにも必要だと感じています。

plugin-packager はその第一歩として、少なくとも「配布構造を統一する」部分を自動化しました。依存解決はまだ先の話ですが、構造が統一されていれば将来的に上に乗せやすいはずです。

---

## まとめ：cmux エコシステムがもたらすもの

```
Before:
  Claude Code（1セッション、1ディレクトリ、ブラックボックス Agent）

After:
  cmux + using-cmux → AI がターミナルを自由に操作
  + cmux-team       → 複数 AI の並列協調
  + cfork           → 会話の気軽なフォーク
  + cmux-remote     → iPhone からリアルタイム監視
```

根本的に変わるのは、**Claude Code が「カレントディレクトリに閉じた単一セッション」から「複数ワークスペースを横断するマルチエージェント」に進化する**ということです。

- 別リポジトリでの作業を AI に丸投げできる
- サブエージェントの作業をリアルタイムで監視できる
- 会話をフォークして並行検証できる
- 席を離れても iPhone から進捗をチェックできる
- これら全てが、人間が直接操作するのと同じターミナル UI 上で見える

AI エージェントコーディングは「AI に指示を出して結果を待つ」フェーズから、「AI チームと一緒に働く」フェーズに入りつつあります。cmux はそのための基盤になると考えています。

---

## リンク

- [cmux](https://cmux.dev) — ターミナルマルチプレクサ本体
- [using-cmux](https://github.com/hummer98/using-cmux) — Claude Code 向け cmux 操作スキル
- [cmux-team](https://github.com/hummer98/cmux-team) — マルチエージェントオーケストレーション
- [cfork](https://github.com/hummer98/cfork) — 会話フォーク
- [cmux-remote](https://github.com/hummer98/cmux-remote) — iPhone リモートビューア
- [plugin-packager](https://github.com/hummer98/plugin-packager) — スキル配布の自動化
