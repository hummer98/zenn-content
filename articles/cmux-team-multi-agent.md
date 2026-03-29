---
title: "cmux-team v3: AIにAIを管理させるのをやめた話"
emoji: "🏗"
type: "tech"
topics: ["claudecode", "cmux", "ai", "agent", "multiagent"]
published: false
slug: "cmux-team-multi-agent"
---

## はじめに

以前、[cmux エコシステムの紹介記事](https://zenn.dev/hummer/articles/cmux-ecosystem-claude-code)で cmux-team について触れました。あの時点ではまだプロトタイプに近い状態でしたが、その後 **根本的にアーキテクチャを書き換えました**。

変更の核心を一言で言うと：

> **Manager を AI エージェントから TypeScript Daemon に置き換えた。**

「AI がタスクを検出して AI を起動して AI の完了を監視する」という構造を、「**コードがタスクを検出してコードが AI を起動してコードが完了を監視する**」に変えました。AI が担当するのは、実際に判断が必要な場面だけです。

この記事では、なぜそう変えたのか、何が解決されたのか、そして既存のマルチエージェントシステムとの違いを書きます。

:::message
cmux-team は実験的なプロジェクトです。設計判断も運用知見も試行錯誤の途中経過であり、プロダクション品質のフレームワークではありません。
:::

## クイックスタート

```bash
# インストール（Plugin も自動設定される）
npm install -g @hummer98/cmux-team

# cmux 上で Claude Code を起動した状態で
/cmux-team:start
```

Daemon と 3 つの Conductor が立ち上がり、TUI ダッシュボードが表示されます。Master タブに切り替えて、自然言語でタスクを伝えるだけです。

```
You:    ログイン機能を React で作って
Master: タスクを作成しました。
        → Daemon がタスク検出（~10s）→ Conductor-1 に割り当て
        → Conductor-1 が Agent を spawn して実装開始

You:    E2E テストも追加して
Master: タスクを作成しました。
        → Conductor-2 に割り当て（並列実行）

You:    進捗は？
Master: Conductor-1: 実装中（Agent 2/3 完了）
        Conductor-2: テスト作成中
        Task #003 APIリファクタリング: キュー待ち
```

![cmux-team 動作画面](/images/cmux-team-screenshot.png)

## 旧アーキテクチャの問題

旧 cmux-team は 4 層すべてが AI エージェントでした。

```
[User] → [Master (AI)] → [Manager (AI/Sonnet)] → [Conductor (AI)] → [Agent (AI)]
```

Manager は Claude Code のセッションとして Sonnet モデルで動作し、`.team/tasks/` をポーリングしてタスクを検出、Conductor を起動し、完了を監視していました。

動くには動いたのですが、3 つの根本的な問題がありました。

### 1. セマンティックな処理は信頼性が低い

Manager の仕事は本質的に決定論的です。「タスクファイルがあるか見る」「空いている Conductor を探す」「プロンプトを送る」「完了マーカーを確認する」。これらは if 文と for ループで書ける処理であって、LLM に推論させる必要がありません。

にもかかわらず AI にやらせていたため、たまにプロトコルを無視して独自判断で動いたり、タスクファイルのフォーマットを壊したりしました。Haiku では頻繁に、Sonnet でも稀に起きます。**決定論的な処理を非決定論的な手段で実行する**のは、そもそも設計として間違っていました。

### 2. タスクのキューイングがセマンティック依存

旧版では Master が Manager に「このタスクをやって」と自然言語で伝え、Manager がそれを解釈して Conductor に振り分けていました。つまり **タスクの受付自体が AI の処理能力に依存**していて、Manager が前のタスクの監視で忙しいときは新しいタスクを受け付けられない。

複数セッションから同時にタスクを投げたい、AI の処理を待たずにひたすらタスクを積みたい、というニーズに対して、AI ベースのキューは構造的に対応できませんでした。

### 3. 全体の状態が見えない

「今どのタスクがどこまで進んでいるか」を知るには、Manager の画面を読むか、ファイルを漁るしかありませんでした。Claude Code のセッションを並べるだけだと、**どのペインが何のタスクをやっているのか**が一目でわかりません。

## 新アーキテクチャ：決定論的 Daemon + セマンティック Agent

v3 では Manager を TypeScript/Bun の常駐プロセス（Daemon）に置き換えました。

![cmux-team v3 アーキテクチャ図](/images/cmux-team-v2-architecture.jpeg)

### 決定論とセマンティクスの境界線

この設計で最も重要なのは、**どこに決定論とセマンティクスの境界線を引くか**です。

| 処理 | 性質 | 担当 |
|------|------|------|
| タスクファイルの検出 | 決定論的 | Daemon |
| 空き Conductor の割り当て | 決定論的 | Daemon |
| プロンプトの送信 | 決定論的 | Daemon |
| Agent 状態の検出 | 決定論的 | Daemon |
| ダッシュボード表示 | 決定論的 | Daemon |
| タスクの分解と設計判断 | セマンティック | Conductor (AI) |
| コードの実装・テスト | セマンティック | Agent (AI) |
| ユーザーとの対話 | セマンティック | Master (AI) |

パイプの接続はコードがやる。パイプの中を流れる判断は AI がやる。この分離により、オーケストレーション層の信頼性が劇的に上がりました。

Kubernetes に例えると、kube-scheduler が Pod をノードに割り当てる処理は決定論的なコードですが、Pod 内のアプリケーションが何をするかは自由。cmux-team の Daemon と Agent の関係もこれと同じです。

## CLI ベースの非同期タスクキュー

旧版で最も不満だった「AI の処理を待たないとタスクを追加できない」問題は、CLI 一発でタスクを追加できる仕組みで解決しました。

```bash
# Master セッションから
cmux-team create-task --title "ログイン機能を実装" --priority high

# 別のターミナルからでも
cmux-team create-task --title "E2Eテストを追加" --priority medium

# どこからでも、何個でも、即座に追加できる
```

タスクは `.team/tasks/NNN-slug.md` としてファイルに書き出されます。タスクの追加はファイルを書くだけなので、どのセッションからでも、AI の処理を待たずに実行できます。Daemon は 10 秒間隔でポーリングして新しいタスクを検出し、空いている Conductor に割り当てます。

状態遷移もシンプルです：

```
draft → ready → in_progress → closed → archived
```

依存関係（`depends_on`）もサポートしており、先行タスクが完了するまで後続タスクは `ready` のままキューに留まります。

## 固定レイアウトと Conductor 再利用

旧版では Conductor をタスクごとに動的に spawn していましたが、v3 では **固定 2x2 レイアウトに 3 つの常駐 Conductor** を配置する方式に変更しました。

```
┌──────────────────┬──────────────────┐
│ Manager(Daemon)  │                  │
│ + Master(tab)    │  Conductor-1     │
│                  │                  │
├──────────────────┼──────────────────┤
│                  │                  │
│  Conductor-2     │  Conductor-3     │
│                  │                  │
└──────────────────┴──────────────────┘
```

Conductor は Claude Code のセッションとして常駐し、タスクが来たら作業し、完了したら `/clear` されてアイドル状態に戻ります。新しいタスクが来たら同じセッションが再利用されます。

```
idle → running → done → idle → running → ...
```

Agent は Conductor のペイン内に**タブとして** spawn されます。ペインを増やさないのでレイアウトが崩れません。

この方式のメリットは：

- **起動コストの削減** — Conductor のセッション確立を毎回待つ必要がない
- **レイアウトの安定** — ペイン数が固定なので画面が崩れない
- **並列度の明示** — 最大 3 タスク並列。4 つ目以降はキューで待機。予測可能

## TUI ダッシュボード

Daemon には React/Ink で実装された TUI ダッシュボードが組み込まれています。

```
╔══════════════════════════════════════════════════════════════════╗
║ cmux-team                              RUNNING  PID:12345      ║
║ Poll: 10s  Conductors: 3/3  Tasks: 2 running / 1 ready        ║
╠══════════════════════════════════════════════════════════════════╣
║ Master: surface:42 ✓                                           ║
╠══════════════════════════════════════════════════════════════════╣
║ Conductors                                                     ║
║ ┌─ conductor-1 [RUNNING] Task #001 "ログイン機能"  (5m23s)     ║
║ │  ├─ agent-impl-1 [DONE]                                      ║
║ │  ├─ agent-impl-2 [RUNNING]                                   ║
║ │  └─ agent-reviewer [WAITING]                                 ║
║ ┌─ conductor-2 [RUNNING] Task #002 "E2Eテスト"     (2m10s)     ║
║ │  └─ agent-tester-1 [RUNNING]                                 ║
║ ┌─ conductor-3 [IDLE]                                          ║
╠══════════════════════════════════════════════════════════════════╣
║ Tasks                                                          ║
║  #001  ログイン機能を実装       in_progress  5m23s              ║
║  #002  E2Eテストを追加          in_progress  2m10s              ║
║  #003  APIリファクタリング      ready        —                  ║
╠══════════════════════════════════════════════════════════════════╣
║ [1] Journal  [2] Log                                           ║
║ 14:23:05 Task #001 assigned to conductor-1                     ║
║ 14:23:07 conductor-1: spawned agent-impl-1 (implementer)       ║
║ 14:25:30 conductor-1: agent-impl-1 done                        ║
║ 14:25:32 conductor-1: spawned agent-impl-2 (implementer)       ║
╚══════════════════════════════════════════════════════════════════╝
```

これが「現在何がどこまで進んでいるのか」の答えです。タスクの状態、Conductor の稼働状況、Agent のツリー、イベントログが一画面で見えます。2 秒間隔で自動更新されるので、**ターミナルを見ているだけで全体の進捗がわかります**。


## 末端エージェントの自由度

cmux-team の設計で意識したのは、**オーケストレーション層だけを提供し、末端の Agent には何も強制しない**ということです。

Agent が知っている情報は「作業内容」と「出力先ファイル」だけです。cmux-team のアーキテクチャも、タスク管理の仕組みも、上位層の存在すら知りません。Agent はただ作業して、終わったら止まる。上位の Daemon が `cmux list-status` API でステータスをポーリングして完了を検出する pull 型の通信なので、Agent 側に報告プロトコルの実装は不要です。

旧版ではこの完了検出も `cmux read-screen` で画面を読み取り「`❯` プロンプトが出たか」をパターンマッチするセマンティックな処理でした。v3 では cmux が提供する `list-status` API に置き換え、**完了検出まで含めて完全に決定論化**しました。

つまり、プリセットの Implementer / Researcher / Tester を使う必要はなく、**あなたが普段使っている Claude Code のセッションや CLAUDE.md の設定をそのまま Agent として組み込めます**。Gemini CLI でも動きます。Daemon が見ているのは cmux API が返すステータスだけだからです。

ガチガチのワークフロー定義は不要で、Conductor に独自のプロンプトを渡すだけで振る舞いを変えられます。必要ならばワークフローを強いることもできるし、何も指定しなければ Conductor が自律的に判断する。この柔軟性はラッパー構造だからこそ実現できます。

## 既存システムとの比較

### 「AI が AI を管理する」系

[Antigravity](https://github.com/antigravity-official/antigravity)、[Auto-Claude](https://github.com/mbenezit/auto-claude)、[multi-agent-shogun](https://github.com/yohey-w/multi-agent-shogun)、Claude Code の組み込み Agent ツールなどは、いずれも **オーケストレーション自体を AI に任せる** アプローチです。

メリットは柔軟性。デメリットはオーケストレーション自体が非決定論的になること。「タスクの振り分けを間違える」「完了検出を見落とす」「プロトコルを無視して暴走する」といった問題が、AI の層が深くなるほど発生確率が上がります。

cmux-team は **オーケストレーション層を決定論的なコードに置き換える** ことで、この問題を構造的に排除しました。AI が判断すべき部分（タスクの分解、コードの実装）だけを AI に任せます。

### SaaS 型エージェント（Devin, Manus, Codex）

[Devin](https://devin.ai/)、[Manus](https://manus.im/)、[OpenAI Codex](https://openai.com/index/codex/) などは完成度の高いプロダクトですが、共通して **実行環境がベンダーのサンドボックス** です。

cmux-team は自分のターミナルで動きます。既存の開発環境、shell 設定、社内ネットワーク、認証情報がそのまま使えます。モデルの入れ替えはテンプレート 1 行の変更で済みます。

### CI/CD パイプライン的アプローチ

GitHub Actions のような「ステップを定義して順番に実行する」方式は決定論的で信頼性が高いですが、**各ステップの内容を事前に定義する必要がある**のが AI エージェントとの相性が悪い点です。

cmux-team は「ルーティングは決定論的、実行はセマンティック」という中間地点を狙っています。タスクの割り当てと完了検出は確実にコードが行い、タスクの中身は AI が自由に判断する。

## おわりに

cmux-team v3 で得た最大の学びは、**AI にやらせるべきことと、コードでやるべきことの境界線は思ったより明確だ**ということです。

既存のマルチエージェントシステムの多くは「AI が AI を管理する」構造を採っています。これは「AI は汎用だから何でもできるはず」という期待に基づいていますが、実際に運用すると、オーケストレーション層に AI を使うことのコストと不安定さが目立ちます。

ファイルの存在確認、プロセスの起動、画面のポーリング、状態の更新。これらは for ループと if 文で書ける処理です。ここに LLM の推論を挟む理由はありません。AI が輝くのは、タスクの分解、コードの設計判断、実装、テストケースの設計といった、**本当に判断が必要な場面**です。

「どこまでをコードで書き、どこからを AI に任せるか」。この境界線の引き方が、マルチエージェントシステムの信頼性を決めると考えています。

## リンク

- [cmux-team リポジトリ](https://github.com/hummer98/cmux-team)
- [cmux](https://cmux.dev)
- [using-cmux](https://github.com/hummer98/using-cmux) -- Claude Code 向け cmux 操作スキル
- [cmux エコシステム記事](https://zenn.dev/hummer/articles/cmux-ecosystem-claude-code)
