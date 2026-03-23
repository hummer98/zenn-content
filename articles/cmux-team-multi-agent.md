---
title: "cmux-team: 4層アーキテクチャで実現するマルチエージェント開発"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "cmux", "ai", "agent", "multiagent"]
published: false
slug: "cmux-team-multi-agent"
---

## はじめに

**最初に断っておくと、cmux-team は実験的なプロジェクトです。** 「cmux で自律実行型のマルチエージェントを組んだらどうなるか？」を検証するために作ったもので、プロダクション品質のフレームワークではありません。設計判断も運用知見も、試行錯誤の途中経過です。

それを踏まえた上で、この記事では [cmux-team](https://github.com/hummer98/cmux-team) の v2.0.0 アーキテクチャと、実際に動かして得た知見を共有します。

> この記事は [cmux エコシステム記事](https://zenn.dev/hummer/articles/cmux-ecosystem-claude-code) の cmux-team セクションを独立させたものです。

## なぜ作ったのか

### 既存ツールの課題感

マルチエージェント開発ツールはいくつか試しました。[Antigravity](https://github.com/antigravity-official/antigravity)、[Auto-Claude](https://github.com/mbenezit/auto-claude)、Claude Code の組み込み Agent ツールによるチーム構成など。どれも可能性は感じたのですが、共通して引っかかるポイントがありました。

- **サブエージェントのブラックボックス化** — Agent ツールで起動したサブエージェントが何をやっているか見えない。実際に途中で介入することは稀ですが、「何をやろうとしているのか」がリアルタイムで見えるだけで安心感が全然違います。結果だけ返ってきて「ファイルを30個変更しました」と言われても、心穏やかではいられません
- **セッション間・プロジェクト間の通信がつらい** — Claude Code はカレントディレクトリドリブンなので、別リポジトリとの連携は「cd して新セッション立ち上げて」の手動作業。自己操作性（あるセッションが別セッションに指示を出す）にも制約がある
- **新規プロジェクトの立ち上げが手動** — いずれもプロジェクト/ディレクトリドリブンなので、実験用のリポジトリを用意したり、ワークスペースを準備したりする作業が人間の手に残る

ghostty + tmux でマルチプロジェクト運用していた時期もありましたが、ターミナルが増えると**今見ているペインが一体どのプロジェクトのどのタスクなのか**がわからなくなります。タブタイトルを手動で管理するのも限界でした。

### インスピレーション

[multi-agent-shogun](https://github.com/yohey-w/multi-agent-shogun) に出会ったのが直接のきっかけです。エージェント階層をターミナル上で可視化しながら自律協調させるというアプローチに「これだ」と感じました。自分でも cmux ベースで同じことをやってみたくなって、cmux-team を作り始めました。

## 設計思想：CLI ベースでベンダーロックインを避ける

cmux-team のエージェント呼び出しは**すべて CLI ベース**です。Manager が Conductor を起動するとき、やっていることは `cmux send` でターミナルにコマンド文字列を送っているだけ。

```bash
cmux send --surface "$SURFACE" "claude --dangerously-skip-permissions '...'\n"
```

この設計を選んだ最大の理由は、**実務を担うエージェントが特定のベンダーにロックインされないようにしたかった**からです。

上の例は Claude Code を起動していますが、ここを差し替えれば Gemini CLI でも OpenAI Codex でも動きます。Manager と Conductor が見ているのは「ターミナルの画面に `❯` が表示されたかどうか」だけなので、下で動いているエージェントが何であるかを気にしません。

```bash
# Claude の代わりに Gemini CLI を起動する例
cmux send --surface "$SURFACE" "gemini '...'\n"
```

API 経由の直接呼び出しではなく、CLI を介した間接呼び出しにすることで、エージェントの入れ替えがターミナルコマンド1行の変更で済みます。

## 何が変わったか

cmux-team（と using-cmux）を導入して、開発ワークフローにいくつかの変化がありました。

### Markdown 地獄からの解放

これは cmux-team というより [using-cmux](https://github.com/hummer98/using-cmux) の恩恵ですが、**他の surface の画面を AI のコンテキストとして直接読める**ようになったのが大きい。

以前は、あるエージェントの状態を別のエージェントに伝えるために、スナップショット情報を Markdown ファイルに書き出しては読ませる、という手順が必要でした。状態が変わるたびにファイルを更新して、読み込み側にも「ファイルを読み直して」と指示する。この Markdown 地獄から解放されました。

```bash
# 別 surface の画面を直接読むだけ
cmux read-screen --surface surface:N --scrollback 200
```

スナップショットではなく、リアルタイムの画面そのものがコンテキストになります。

### 作業キューによるプロジェクト多面打ち

Master にタスクを立て続けに指示できるようになったことで、**「次のタスクの指示」のために作業完了を待つ時間が減りました**（ゼロではないですが）。

```bash
# Master にタスクを連続投入
"ログイン機能を実装して"
"テストを書いて"
"ドキュメントを更新して"
```

タスクファイルがセマンティックな作業キューとして機能するので、1プロジェクトへの指示をまとめて行えます。結果として、複数プロジェクトを同時進行する「多面打ち」のインターバルを伸ばせるようになりました。プロジェクト間のコンテキストスイッチ回数が減り、人間側の認知負荷が下がります。

### 非決定論的タスクの性能比較

複数のリポジトリ/ディレクトリの作成から実験結果の統合までを完全自律化できるので、**セマンティックな挙動の確認など、決定論的でないタスクの性能比較を気軽にできる**ようになりました。

例えば「プロンプト A と B でエージェントの出力品質がどう変わるか」を検証したいとき、それぞれの実験環境を自動構築して並列実行し、結果を統合するところまでを cmux-team に任せられます。

ただしトレードオフもあって、**トークン消費は爆発します**。マルチエージェントで並列実行すると、単体実行の数倍〜十数倍のトークンを消費するのは覚悟してください。

## 4層アーキテクチャ

cmux-team の核は、4つの層に分離されたエージェント階層です。

```
Master（ユーザーと対話、task を作成）
  └── Manager（アイドル待機、task 検出 → Conductor 起動）
        └── Conductor（git worktree でタスクを自律実行）
              └── Agent（実作業: 実装・テスト・リサーチ等）
```

### 各層の責務

| 層 | 責務 | ライフサイクル |
|----|------|---------------|
| **Master** | ユーザーの指示を `.team/tasks/open/` にタスクファイルとして書き出す。進捗をユーザーに報告する。 | ユーザーセッション中は常駐 |
| **Manager** | タスクを検出して Conductor を起動。pull 型で進捗を監視し、完了したら結果を回収してタスクをクローズ。 | アイドル停止 + イベント駆動 |
| **Conductor** | 1タスクを git worktree 内で自律完遂。必要に応じて Agent を起動して並列作業。 | タスク完了で停止 |
| **Agent** | 実際のコード実装・テスト・リサーチを行う。 | 完了したら停止 |

### なぜ4層か

旧版では Conductor が直接サブエージェントを束ねるフラットな構造でしたが、2つの問題がありました。

1. **Conductor がボトルネックになる** — タスク管理と実作業を兼任すると、並列タスクのスケジューリングが破綻する
2. **ユーザーとの対話が途切れる** — Conductor が作業に集中すると、ユーザーへの報告が遅れる

4層に分離したことで、各層が「自分の仕事だけ」に集中できるようになりました。Master は作業しない。Manager は監視だけ。Conductor は1タスクだけ。Agent は実作業だけ。

## イベント駆動の Manager

### 常駐ループをやめた理由

初期の Manager は常駐ループで `.team/tasks/open/` を走査していました。30秒おきにファイルシステムをチェックするポーリングです。

これには問題がありました。

- **無駄な API 消費** — タスクがないのにループが回り続ける
- **コンテキストの肥大化** — ループのたびに LLM のコンテキストが膨らむ
- **レート制限への圧力** — 複数エージェントが同時に動くので、Manager のポーリングが追い打ちをかける

### `[TASK_CREATED]` 通知で起床

v2.0.0 の Manager は**アイドル停止 + イベント駆動**です。

```
1. Master がタスクを作成 → .team/tasks/open/ にファイルを置く
2. Master が Manager に [TASK_CREATED] 通知を送信 → cmux send
3. Manager が起床 → タスクを読み込み → Conductor を起動
4. Conductor 完了を検出 → 結果回収 → タスククローズ
5. 未処理タスクがなければアイドル停止
```

Manager は「待っている間は何もしない」。タスクが来たら起きて処理し、終わったらまた停止する。API 呼び出しの無駄がなくなりました。

### Sonnet モデルの選定

Manager は **Sonnet モデル**で動作します。これは試行錯誤の結果です。

- **Haiku**: 指示に従わなかった。ループプロトコルを無視して独自の判断で動いたり、タスクファイルのフォーマットを守らなかったりした
- **Opus**: 動作するが、Manager の仕事（タスク検出 → Conductor 起動 → 監視 → 回収）には過剰。コストパフォーマンスが悪い
- **Sonnet**: 指示に忠実に従いつつ、異常時のリカバリ判断もできる。Manager のタスクに最適

## spawn-conductor.sh — 決定論的な起動

### LLM 判断の揺れを排除する

Conductor の起動には、以下の手順が必要です。

1. git worktree を作成
2. worktree のブートストラップ（`npm install` 等）
3. ペインを作成
4. プロンプトを生成
5. Claude を起動
6. Trust 確認を自動承認

これらは **毎回同じ手順** です。LLM に判断させる余地がない。にもかかわらず LLM に実行させると、手順を飛ばしたり順序を間違えたりすることがありました。

### スクリプト化のメリット

`.team/scripts/spawn-conductor.sh` にスクリプト化することで、起動手順が**決定論的**になりました。

```bash
# Manager がスクリプトを呼び出すだけ
bash .team/scripts/spawn-conductor.sh \
  --task ".team/tasks/open/001-implement-login.md" \
  --project-root "$(pwd)"
```

スクリプト内では:

```bash
# 1. git worktree 作成
CONDUCTOR_ID="conductor-$(date +%s)"
git worktree add ".worktrees/${CONDUCTOR_ID}" -b "${CONDUCTOR_ID}/task"

# 2. ブートストラップ
cd ".worktrees/${CONDUCTOR_ID}"
[ -f package.json ] && npm install
cd -

# 3. プロンプト生成（テンプレートから変数置換）
# 4. ペイン作成
SURFACE=$(cmux new-split down)
cmux rename-tab --surface "$SURFACE" "[C] ${CONDUCTOR_ID}"

# 5. Claude 起動
cmux send --surface "$SURFACE" \
  "claude --dangerously-skip-permissions \
   '.team/prompts/${CONDUCTOR_ID}.md を読んで指示に従って作業してください。'\n"

# 6. Trust 確認の自動承認
for i in $(seq 1 10); do
  SCREEN=$(cmux read-screen --surface "$SURFACE" 2>&1)
  if echo "$SCREEN" | grep -q "Yes, I trust"; then
    cmux send-key --surface "$SURFACE" "return"
    sleep 3; break
  elif echo "$SCREEN" | grep -qE '(Thinking|Reading|❯)'; then
    break
  fi
  sleep 3
done
```

この設計の背景にあるのは「**決定論的な処理はコードで、判断が必要な処理は AI で**」という原則です。起動手順のような定型作業を LLM に任せると、失敗モードが予測不能になります。

## Pull 型通信と真のソース参照

### すべて pull 型

cmux-team の通信は**すべて pull 型**です。下位層は完了したら停止するだけ。上位層が `cmux read-screen` で画面を読んで完了を検出します。

```
Master → Manager:  .team/tasks/open/ にファイルを置く + [TASK_CREATED] 通知
Manager → Conductor:  cmux send でプロンプト送信
Manager ← Conductor:  cmux read-screen で完了検出
Conductor → Agent:  cmux send でプロンプト送信
Conductor ← Agent:  cmux read-screen で完了検出
```

完了判定は画面のプロンプト表示で行います。

```bash
SCREEN=$(cmux read-screen --surface surface:N --lines 10 2>&1)

# ❯ が表示 AND "esc to interrupt" がない → 完了（アイドル状態）
# ❯ が表示 AND "esc to interrupt" がある → まだ実行中
```

### status.json の廃止

以前は Manager が `.team/status.json` を更新し、Master がそれを読んで進捗を報告していました。しかし status.json には問題がありました。

- **更新タイミングのずれ** — Manager がファイルを書き出すタイミングと、実際の状態が一致しない
- **Manager の負担** — ループのたびに JSON を組み立てて書き出す処理が、コンテキストを無駄に消費する
- **二重管理** — タスクファイルと status.json の両方に状態が書かれ、矛盾が生じる

v2.0.0 では status.json を廃止し、**Master が真のソースを直接参照する**設計に変更しました。

```bash
# Master が直接確認する
ls .team/tasks/open/   # 未処理タスク
ls .team/tasks/closed/ # 完了タスク
ls .team/output/       # Conductor の出力
```

タスクファイル自体が真のソースです。中間的な status.json を経由する必要がなくなりました。

### Agent は上位を気にしない

この設計の重要なポイントは、**下位の Agent が上位との協調動作を一切気にしなくてよい**ことです。上位が polling で見に来るので、Agent 側は「作業して、終わったら止まる」だけ。

つまり、cmux-team の本質は特定のロール群ではなく、**既存の Agent やツールを自由に組み込んで、ワークフローをキック・監視する仕組み**です。

## git worktree による隔離

各 Conductor は **git worktree** で独立したブランチを持ちます。main ブランチは常に無傷。

```bash
# Manager が Conductor 用の worktree を作成
git worktree add .worktrees/conductor-1 -b conductor-1/task

# Conductor と Agent はすべてこの中で作業
# 完了後、Manager が main にマージして worktree を削除
```

### ブートストラップの重要性

git worktree は tracked files のみチェックアウトします。`.gitignore` されたディレクトリ（`node_modules/`, `dist/` 等）は手動で再構築する必要があります。

```bash
cd .worktrees/conductor-N
[ -f package.json ] && npm install
[ -f .envrc ] && direnv allow
```

これを忘れると、Conductor が「依存関係がない」というエラーで即座にクラッシュします。spawn-conductor.sh にブートストラップを組み込んでいるのは、この失敗を防ぐためです。

## 構造層とプリセットロール

cmux-team で固定なのは **Manager**（監視 + タスクディスパッチ）と **Conductor**（タスク実行 + Agent 管理）の2つだけです。

その下で実際に動く Agent には、以下のプロンプトテンプレートがプリセットとして同梱されています。

| ロール | 用途 |
|--------|------|
| Researcher | 技術調査・情報収集 |
| Architect | 設計・アーキテクチャ決定 |
| Implementer | コード実装 |
| Tester | テスト作成・実行 |
| Reviewer | コードレビュー |
| DocKeeper | ドキュメント管理 |
| IssueManager | イシュー整理・優先度付け |

**これらは出発点にすぎません。** 自分のプロジェクトで使い慣れた Agent やツールがあれば、テンプレートを差し替えてそのまま組み込めます。

## スラッシュコマンド

コマンドは「基本コマンド」と「手動オーバーライド」に分類されています。

**基本コマンド**（4層アーキテクチャで動作）:

```
/start            → Master + Manager 起動
/team-status      → タスク状態の表示
/team-disband     → 全層を bottom-up で終了
/team-spec        → 要件ブレスト（Master が直接対話）
/team-task        → タスクの作成・一覧・クローズ
```

**手動オーバーライド**（Manager を経由せず直接実行）:

```
/team-research    → 並列リサーチ
/team-design      → 設計 + レビュー
/team-impl        → 並列実装
/team-review      → コードレビュー
/team-test        → テスト実行
/team-sync-docs   → ドキュメント同期
```

手動オーバーライドは、4層を立ち上げるほどではない小さなタスクに便利です。Master に指示を出す代わりに、直接 Agent を起動して作業させます。

## 実際の運用で得た知見

### 権限制限の課題

Claude Code の `--dangerously-skip-permissions` は全ツールの権限チェックをスキップします。サブエージェントを自動起動するには必須ですが、権限の粒度が粗いのが悩みです。

理想的には「ファイル読み書きは許可、外部 API 呼び出しは都度確認」のような細かい制御がほしいところです。現状は `--dangerously-skip-permissions` を付けて、CLAUDE.md や Agent プロンプトで制約を書くしかありません。

### Haiku が指示に従わない問題

Haiku で Manager を動かした際、以下の問題が発生しました。

- ループプロトコルを無視して、タスクを自分で実行しようとする
- タスクファイルのフォーマットを守らない（frontmatter を省略する等）
- Conductor の完了検出ロジックを独自に変更する

Sonnet に変更したところ、これらの問題はすべて解消しました。Manager のような「プロトコルに忠実に従う」ロールには、指示追従性の高いモデルが適しています。

### Trust 確認の自動化

Claude Code を `--dangerously-skip-permissions` で起動すると、初回に Trust 確認ダイアログが表示されます。これを自動承認するために、`cmux read-screen` で画面を監視して `cmux send-key return` で Enter を送信しています。

```bash
# Trust 確認検出 → 自動承認
for i in $(seq 1 10); do
  SCREEN=$(cmux read-screen --surface "$SURFACE" 2>&1)
  if echo "$SCREEN" | grep -q "Yes, I trust"; then
    cmux send-key --surface "$SURFACE" "return"
    sleep 3; break
  fi
  sleep 3
done
```

タイミング依存なので、稀に手動で承認が必要になることがあります。

### レイアウト戦略

Agent を増やすとペインが狭くなります。`cmux new-split right` だけで横に並べると、1/2 → 1/4 → 1/8 と指数的に縮小します。

**`right` と `down` を組み合わせてグリッド配置する**のがコツです。

```
[Master    ] | [Manager  ]
[Conductor ] | [Agent A  ]
```

7ペイン以上は1つのワークスペースに収まらないので、`cmux new-workspace` でワークスペースを分けます。ただし `new-workspace` には後述の PTY 遅延初期化問題があるため、現状は同一ワークスペース内の `new-split` を優先しています。

### cmux 本体の未解決問題

cmux-team の運用で遭遇している cmux 本体側の issue を2つ紹介します。どちらもワークアラウンドで回避していますが、正直なところ力技です。

#### PTY 遅延初期化問題（[#1472](https://github.com/manaflow-ai/cmux/issues/1472)）

`cmux new-workspace` でプログラム的に作成したワークスペースのターミナル PTY は、**GUI 上で一度表示されるまで起動しない**という問題です。

バックグラウンドのワークスペースでは PTY（`ghostty_surface_t`）が生成されないため、`cmux read-screen` は `Surface not found` エラー、`cmux send-key` は `Surface not ready` エラーになります。`cmux send` は OK を返しますが、テキストはキューに留まって実際には配信されません。

ワークアラウンドとして、**AppleScript でメニュークリックしてワークスペースを GUI 描画させてから元に戻す**という荒業をやっています。

```bash
WS=$(cmux new-workspace --cwd $(pwd) | awk '{print $2}')

# ワークスペースのインデックスを取得
WS_INDEX=$(cmux tree --json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for w in data['windows']:
    for ws in w['workspaces']:
        if ws['ref'] == '$WS':
            print(ws['index'] + 1)")

# AppleScript でメニュークリック → GUI 描画を強制 → PTY 初期化
osascript -e "
tell application \"System Events\"
    tell process \"cmux\"
        click menu item \"ワークスペース $WS_INDEX\" of menu 1 of menu bar item \"表示\" of menu bar 1
    end tell
end tell"
sleep 2

# 元のワークスペースに戻る
osascript -e "
tell application \"System Events\"
    tell process \"cmux\"
        click menu item \"ワークスペース 1\" of menu 1 of menu bar item \"表示\" of menu bar 1
    end tell
end tell"
```

macOS のアクセシビリティ許可が必要で、画面もチラつきます。cmux 本体側でヘッドレスな PTY 生成がサポートされれば不要になるはずですが、現状はこの方法しかありません。

この問題があるため、cmux-team では `new-workspace` よりも同一ワークスペース内の `new-split` を優先するレイアウト戦略を採っています。

#### drag-surface-to-split の cross-workspace 問題（[#1901](https://github.com/manaflow-ai/cmux/issues/1901)）

`cmux drag-surface-to-split` は V1 ソケットハンドラを経由しており、surface の解決に**現在 GUI でフォーカスされているワークスペース**を使います。そのため、呼び出し元と異なるワークスペースの surface を操作しようとすると `Surface not found` エラーになります。

`new-split` などの他のコマンドは V2 ハンドラで `--workspace` パラメータを受け付けるため cross-workspace で動作しますが、`drag-surface-to-split` だけが取り残されている状態です。

ワークアラウンドとしては、**`drag-surface-to-split` を使わずに `new-split` + `cmux send` で代替する**か、操作前に `select-workspace` で対象ワークスペースにフォーカスを移してから実行しています。

## インストール

```bash
# Plugin（推奨 — スキル + コマンド + フック全対応）
/plugin marketplace add hummer98/cmux-team
/plugin install cmux-team@hummer98-plugins

# Agent Skills（スラッシュコマンドなし、自然言語のみ）
npx skills add hummer98/cmux-team
```

cmux-team は [cmux](https://cmux.dev) と [using-cmux](https://github.com/hummer98/using-cmux) が前提です。cmux セッション内で Claude Code を起動してから使ってください。

---

## リンク

- [cmux-team リポジトリ](https://github.com/hummer98/cmux-team)
- [cmux](https://cmux.dev) — ターミナルマルチプレクサ本体
- [using-cmux](https://github.com/hummer98/using-cmux) — Claude Code 向け cmux 操作スキル
- [cmux エコシステム記事](https://zenn.dev/hummer/articles/cmux-ecosystem-claude-code) — 全体概要
