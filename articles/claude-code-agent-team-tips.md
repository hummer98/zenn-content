---
title: "Claude Code で AI オーケストレーションを自作したい人向けの TIPS"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "multiagent", "proxy", "ai", "agent"]
published: true
slug: "claude-code-agent-team-tips"
---

## はじめに

[cmux-team](https://github.com/hummer98/cmux-team) を作っていく過程で、Claude Code を複数プロセス起動して協調させるシステムを自作すると、公式ドキュメントには書いていないがハマりやすいポイントがいくつかある。

この記事は「Claude Code で AI オーケストレーション基盤を自作したい人」向けの実装 TIPS 集です。

:::message
cmux-team 固有の話も含みますが、`ANTHROPIC_BASE_URL` プロキシやセッション管理など、Claude Code で AI オーケストレーションを組む全般に応用できる内容です。
:::

## プロキシを挟むと世界が広がる

### トークン残量を知る

Claude Code のトークン残量を取得したい場合、**`ANTHROPIC_BASE_URL` に自前のプロキシを指定する**のが唯一の実用的な手段です。OTELメトリクスでは取得できません。

```bash
export ANTHROPIC_BASE_URL=http://localhost:3000
```

あとはプロキシ側で Anthropic API のレスポンスヘッダを覗くだけです。

#### Claude Max なら `unified-*` ヘッダを見る

Claude Max サブスクリプションで Claude Code を走らせると、レスポンスには以下のヘッダが返ってきます。

```
anthropic-ratelimit-unified-status: allowed | rate_limited
anthropic-ratelimit-unified-5h-utilization: 0.018
anthropic-ratelimit-unified-5h-reset: 1776523064
anthropic-ratelimit-unified-7d-utilization: 0.737
anthropic-ratelimit-unified-7d-reset: 1776783264
```

- `5h` / `7d` はそれぞれ 5 時間バースト制限と 7 日間ウィークリーキャップ
- `utilization` は 0.0〜1.0 の使用率（1.0 で枯渇）
- `reset` は unix epoch 秒で、その時刻に窓がリセットされる
- `status` が `rate_limited` になったらそのまま使えばほぼ確実に 429 が返る

cmux-team では utilization が一定閾値（デフォルト 0.95）を超えたら子エージェントの起動をスロットルする、という制御に使っています。

#### API プラン（従来プラン）は `tokens-*` ヘッダ

従来の TPM（tokens per minute）制限で動くプランでは別ヘッダが返ります。

```
anthropic-ratelimit-tokens-remaining: 394000
anthropic-ratelimit-tokens-limit: 400000
anthropic-ratelimit-tokens-reset: 2026-04-19T12:34:56Z
anthropic-ratelimit-input-tokens-remaining: 197000
anthropic-ratelimit-output-tokens-remaining: 79000
```

両系統とも**同時に返ることはあまりない**ため、プロキシ側は「unified が取れなければ tokens-* にフォールバック」する実装にしておくと無難です。

#### `reset` は epoch 秒と ISO 8601 の両方がありうる

ヘッダの値は **数値（epoch 秒）で返る場合と ISO 8601 文字列で返る場合**があるので、両方を吸収するノーマライザを噛ませるのが安全です。cmux-team では以下のような関数で統一しています。

```ts
function toEpochSec(raw: string | null): number | null {
  if (!raw) return null;
  const asNum = Number(raw);
  if (!isNaN(asNum) && asNum > 1e9) return Math.floor(asNum); // epoch 秒
  const ms = new Date(raw).getTime();                          // ISO 8601
  return isNaN(ms) ? null : Math.floor(ms / 1000);
}
```

「次に使えるのはいつか」をユーザに見せるときはここで秒数に正規化してから差分を取ると、表示ロジックが綺麗になります。

#### `CLAUDE_CODE_OAUTH_TOKEN` 運用でも `unified-*` ヘッダは取れる

CI や daemon 常駐用に `CLAUDE_CODE_OAUTH_TOKEN` 環境変数で認証する運用は便利ですが、**この状態だと Claude Code の `/usage` スラッシュコマンドが動きません**（対話ログイン時のみ機能する仕様）。クォータを視覚的に確認する手段がなくなるので、普段 `/usage` に頼っていると詰まります。

ただし **API レスポンスの `anthropic-ratelimit-unified-*` ヘッダは `CLAUDE_CODE_OAUTH_TOKEN` 経由でも通常通り返ってきます**。したがって自前プロキシさえ挟めば、OAuth token 認証の常駐環境でも 5h / 7d の utilization と reset を拾えます。cmux-team は `.envrc` で `CLAUDE_CODE_OAUTH_TOKEN` を仕込んだ Conductor 群でもこのヘッダからスロットル判定を行っています。

### セッションの素性をヘッダで識別する

子セッションが複数ある環境では、どの子からの API コールかをプロキシ側で区別したくなります。

子プロセス起動時にカスタムヘッダを差し込んでおけば、プロキシで識別できます。Claude Code は `ANTHROPIC_CUSTOM_HEADERS` 環境変数を読んで API リクエストに任意ヘッダを乗せてくれます。複数ヘッダは `\n` 区切りです。

```bash
export ANTHROPIC_CUSTOM_HEADERS=$'x-cmux-task-id: 042\nx-cmux-conductor-id: conductor-1\nx-cmux-role: implementer'
```

`settings.json` の `env` キーでも同等に設定できます。

```json
{
  "env": {
    "ANTHROPIC_CUSTOM_HEADERS": "x-cmux-task-id: 042\nx-cmux-conductor-id: conductor-1\nx-cmux-role: implementer"
  }
}
```

さらに便利な点として、**Claude Code はストリーミングレスポンスの初回に `x-claude-code-session-id` ヘッダを返してきます**。プロキシがこれを拾うと、リクエストを送ってきた子セッションの sessionId を後付けで確定できます。

## セッション管理の罠

### session-id の鶏卵問題

エージェントを自動起動するシステムを書くとき、「起動前に sessionId を確定させたい」ケースがあります（トレース DB に事前登録したい、JSONL パスを事前に知りたいなど）。

解決策は**こちら側で UUID を生成してから `--session-id` オプションで渡す**ことです。

```ts
const sessionId = crypto.randomUUID();
// daemon に事前通知
await fetch(`http://localhost:${proxyPort}/api/messages`, {
  method: 'POST',
  body: JSON.stringify({ type: 'CONDUCTOR_SESSION', sessionId }),
});
// Claude をその UUID で起動
execFile('claude', ['--session-id', sessionId, ...otherArgs]);
```

これで Claude から最初のレスポンスが返るより前に sessionId が確定し、JSONL パス（`~/.claude/projects/<cwd-hash>/<UUID>.jsonl`）も事前に計算できます。

### `claude --resume` は「起動時と同じ cwd」でしか動かない

resume は sessionId 単独ではなく、`~/.claude/projects/<encoded-cwd>/<sessionId>.jsonl` を探します。つまり **セッションを開始した cwd と resume 時の cwd が異なると JSONL が見つからず失敗します**。

git worktree を使うシステムだと特に要注意です。worktree 内で起動したセッションを resume するには、resume 時も同じ worktree パスに入り直す必要があります。

```ts
// sessionId と一緒に worktreePath も必ず永続化しておく
execFile('claude', ['--resume', sessionId, ...], {
  cwd: task.worktreePath,  // 起動時と同じ cwd で入る
});
```

### `/compact` 前の会話を読み返す

`/compact` は破壊的操作で、resume すると compact 後の状態からしか再開できません。ただし**生ログの JSONL は残っています**。

```
~/.claude/projects/<エンコード済みcwd>/<session-id>.jsonl
```

compact しても上書きされず、compact 前のエントリもそのまま残ります。`jq` で user / assistant メッセージを抽出すれば過去の発言を読み返せます。

ただし「見れる」≠「resume できる」です。過去時点から再開したければ JSONL を truncate して別ファイル化する手作業が必要になります。

### `/clear` と `/compact` で session-id が変わる

自作基盤で忘れがちですが、**`/clear` は内部で新しい session-id を発行します**。初回起動時の session-id を永続化したきり追従していないと、`/clear` 後の resume は `No conversation found with session ID: <UUID>` で失敗します。

`/compact` も SessionStart hook を `source=compact` で発火させるため、**new session に切り替わる可能性**を前提に作っておくのが安全です。

対策として、cmux-team では `SessionStart` hook の matcher を `""`（空文字＝全 source 許容）にして、`startup` / `resume` / `clear` / `compact` 全てで hook を発火させ、stdin で渡ってくる `session_id` を daemon 側に push しています。

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "my-daemon send SESSION_STARTED --from-stdin --pid $PPID"
      }]
    }]
  }
}
```

hook の stdin には `{session_id, source, transcript_path, cwd, ...}` の JSON が流れてくるので、これを読み取って最新の session-id に追従すれば `/clear` / `/compact` 後の resume も壊れません。

## プロセス管理

### hook を使い倒す

Claude Code の挙動を外から観測・制御するなら hook が最強です。pull 型の画面ポーリングより遥かに早く状態変化を検出できます。

まず公式ドキュメントは以下にまとまっているので、hook 一覧・ペイロード仕様・exit code の意味はこちらを参照してください。

- **公式: [Hooks Reference](https://code.claude.com/docs/en/hooks)**

以下は **cmux-team を実装する過程で「公式ドキュメントだけ読んでいては気づけなかった」挙動や実装 TIPS** を、各 hook ごとに整理したものです。ペイロード（stdin に渡る JSON）も併記しますが、各 hook とも以下の共通フィールドに加えて固有フィールドが乗る、という構造です。

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/<cwd>/<session_id>.jsonl",
  "cwd": "/path/to/project",
  "permission_mode": "default|plan|acceptEdits|auto|dontAsk|bypassPermissions",
  "hook_event_name": "<event>"
}
```

#### 共通 TIPS 1: matcher の解釈は hook ごとに違う

公式には hook ごとに matcher 仕様が書かれていますが、**全 hook で解釈が統一されていない**点は強調されていません。ざっくり 3 種類あります。

| 種類 | 書き方 | 該当 hook |
|------|-------|-----------|
| 厳密一致 1 値（空文字で全捕捉） | `"startup"`, `""` | SessionStart |
| 正規表現 | `"logout\|prompt_input_exit\|other"` | SessionEnd, PreToolUse, PostToolUse, Notification 等 |
| matcher なし | — | Stop, UserPromptSubmit 等 |

特に **SessionStart で `"startup|resume"` のような正規表現を書いても一切発火しません**。全 source 捕捉は `matcher: ""` が定石。

#### 共通 TIPS 2: stdin の JSON は CLI 本体にそのまま流す

shell 内で `jq` を駆使して session_id だけ取り出す… という実装はエスケープ地獄になります。cmux-team は `cmux-team send <TYPE> --from-stdin` という入口を用意して、stdin の生 JSON を TypeScript + Zod で parse/validate しています。新しい hook に対応するのも case を足すだけで済みます。

#### 共通 TIPS 3: hook のシェル環境変数は「claude 起動時点の env」がそのまま継承される

公式ドキュメントは hook のシェルに入っている環境変数について**ほぼ何も書いていません**。明記されているのは以下だけです。

| 環境変数 | 用途 | 対象 hook |
|---------|------|----------|
| `CLAUDE_ENV_FILE` | 後続の Bash コマンドに env を渡すための一時ファイル | SessionStart / CwdChanged / FileChanged |
| `CLAUDE_PROJECT_DIR` | プロジェクトルート絶対パス | 全 hook |
| `CLAUDE_PLUGIN_ROOT` | プラグインのインストールディレクトリ | プラグイン由来の hook |
| `CLAUDE_PLUGIN_DATA` | プラグインの永続データディレクトリ | プラグイン由来の hook |
| `CLAUDE_CODE_REMOTE` | リモート Web 環境で `"true"`、ローカルでは未設定 | 全 hook |

これ以外のことは全部「Hooks run in the current directory with Claude Code's environment」とだけ書かれていて中身は未定義です。

実測では、**claude プロセスが spawn された瞬間の env が、claude → hook へそのまま継承されます**（OS の子プロセス継承がそのまま効いているだけ）。どう env を注入したかは問いません。

```bash
export MY_VAR=foo && claude        # shell の env → claude → hook
MY_VAR=foo claude                   # インライン env → claude → hook
```

```json
// settings.json の env フィールド → claude → hook
{ "env": { "MY_VAR": "foo" } }
```

```ts
// Node.js から spawn する前に書き換え → claude → hook
process.env.MY_VAR = "foo";
execFile("claude", [...]);
```

いずれも hook 内で `"${MY_VAR}"` で参照できます。cmux-team が hook コマンド内で `"${CMUX_SURFACE}"` や `"$PPID"` を使っているのはこの挙動に依存しています。ただし重要な制約が 2 つあります。

- **claude 起動後に外で `export` しても既存の claude には効かない**。env は起動時点でフリーズしているため、新しい env を反映したい場合はセッションを作り直す必要があります
- **PID や PPID は stdin JSON にも env にも公式には入っていません**。`$PPID` で親（= claude プロセス）の PID が取れているのは hook が bash 経由で起動された結果であり、ドキュメントされていない挙動です。将来の Claude Code のアーキテクチャ変更で壊れる可能性があります

---

### セッションライフサイクル系

#### `SessionStart` — セッション開始・再開

```json
{
  "hook_event_name": "SessionStart",
  "source": "startup|resume|clear|compact",
  "model": "claude-sonnet-4-6",
  "...": "共通フィールド"
}
```

- matcher に `""` を入れると `startup` / `resume` / `clear` / `compact` の **全 source を一発捕捉**
- **`/clear` も `/compact` も session-id が変わる可能性がある**。永続化した初回 session-id を使い回すと resume が壊れるので、この hook で来た `session_id` を常に最新値として保持すること
- `model` フィールドで現在選択中のモデル名が取れる。ロール別の制約を書くときに使える
- `$PPID` で Claude プロセスの PID が取れるので、PID 監視 daemon はここで通知するのが最速

#### `SessionEnd` — セッション終了

```json
{
  "hook_event_name": "SessionEnd",
  "end_reason": "clear|resume|logout|prompt_input_exit|bypass_permissions_disabled|other"
}
```

- **`/compact` では発火しません**。compact はセッションを圧縮するだけで終わらせないため
- matcher は正規表現 OK。`"logout|prompt_input_exit|other"` のような OR 指定が自然
- `end_reason` は stdin に入るので、ハンドラ 1 個に集約して受信側で分岐する手もある

**cmux-team での `end_reason` の扱い分け:**

| `end_reason` | 何が起きた | cmux-team の扱い |
|-------------|-----------|----------------|
| `clear` | `/clear` で会話だけリセット（プロセス継続・session-id 交代） | ロールにより分岐。Conductor は `SESSION_CLEAR` メッセージとして「session 切り替え」扱い。Master は `/clear` 後も生存継続したいので matcher から**除外** |
| `resume` | `--resume` / `--continue` / `/resume` のために一度終了 | matcher に入れない。続けて `SessionStart(resume)` が来るので、そちらで最新 session-id を追従すれば十分 |
| `logout` | ログアウトによる終了 | 「本当に終わった」扱いで `SESSION_ENDED` 送信 |
| `prompt_input_exit` | プロンプト入力が閉じられた（Ctrl+D / `exit`） | 同上 |
| `other` | 実測ではクラッシュや親プロセス kill、予期しない終了 | 同上 |
| `bypass_permissions_disabled` | `--dangerously-skip-permissions` が途中無効化 | 実運用で観測したことがないので `logout|prompt_input_exit|other` と同じ扱いで十分 |

つまり cmux-team では**「session-id 切り替え (`clear`)」「本当の終了 (`logout|prompt_input_exit|other`)」「SessionStart 側で処理すれば済むもの (`resume`)」の 3 バケツ**に分けています。`end_reason` を全部マッチさせたくなりますが、`resume` は SessionEnd 側で拾う意味がほぼありません。

#### `Stop` — 応答完了

```json
{
  "hook_event_name": "Stop",
  "stop_reason": "string",
  "last_assistant_message": "..."
}
```

- matcher なし。ターンが終わるたびに必ず発火。外から polling するより遥かに早く「入力待ちに入った」を検知できる
- `last_assistant_message` でその回の最終発話が拾えるので、**AskUserQuestion で止まったのか単なる idle なのかを判別**できる。cmux-team ではここで分岐して別メッセージを送っている
- **exit 2 を返すと Claude は停止せず応答を継続**（stderr が Claude にフィードバックされる）。「テストが通るまで止まるな」系のガードレールに使える

#### `StopFailure` — API エラーで停止

```json
{
  "hook_event_name": "StopFailure",
  "error_type": "rate_limit|authentication_failed|billing_error|invalid_request|server_error|max_output_tokens|unknown",
  "error_message": "..."
}
```

- Stop と違って**エラーで終わったケース専用**。通常の idle 通知と区別したいときはこっちを見る
- `rate_limit` を捕まえれば「5h キャップに当たった」を即座に検知できる（プロキシのヘッダ監視よりラグが小さい）

#### `UserPromptSubmit` — プロンプト送信

```json
{
  "hook_event_name": "UserPromptSubmit",
  "prompt": "ユーザが送信したテキスト"
}
```

- 入力送信の瞬間に発火。Stop とペアで `running` / `idle` の 2 状態を完全にカバーできる
- `prompt` にテキストが入るので、禁則ワード検査して exit 2 で棄却、のような使い方もできる

---

### ツール実行系

#### `PreToolUse` — ツール実行直前

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash|Edit|Write|...|mcp__*",
  "tool_input": { "command": "...", "file_path": "...", "...": "ツール依存" },
  "tool_use_id": "toolu_..."
}
```

- matcher はツール名正規表現。`"mcp__.*"` で MCP 全部、`"Edit|Write|MultiEdit"` で書き込み系だけ、みたいに絞れる
- `tool_input` に引数が丸ごと入っているので、**Bash なら `tool_input.command` を検査して危険コマンドだけ静かに exit 2** などのガードが書ける
- stdout に `hookSpecificOutput` で `{"permissionDecision": "allow|deny|ask"}` を返すと権限プロンプトを出さずに自動判定できる

#### `PostToolUse` — ツール実行成功後

```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Edit",
  "tool_input": { "file_path": "...", "content": "..." },
  "tool_response": { "filePath": "...", "success": true },
  "tool_use_id": "toolu_..."
}
```

- **成功時のみ**発火。失敗時は `PostToolUseFailure` が別に飛ぶ
- Edit/Write の後に自動 lint/format を回す、といった使い方が定番

#### `PostToolUseFailure` — ツール実行失敗後

```json
{
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": { "command": "..." },
  "tool_use_id": "toolu_...",
  "error": "エラーメッセージ",
  "is_interrupt": true
}
```

- ツール失敗専用。`is_interrupt` は **ユーザが ESC 等で中断した場合 true**。モデルエラーとユーザ中断を区別できる

#### `PermissionRequest` — 権限ダイアログ表示時

```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf ..." },
  "permission_suggestions": [
    { "type": "addRules", "behavior": "allow|deny|ask", "destination": "session|localSettings|projectSettings|userSettings" }
  ]
}
```

- 権限ダイアログを **外から自動応答**できる。`permission_suggestions` に提案が含まれるので、それを採否する形
- CI やリモート Agent で「対話 UI なし」運用するときに使う

#### `PermissionDenied` — オートモードで拒否

```json
{
  "hook_event_name": "PermissionDenied",
  "permission_mode": "auto",
  "tool_name": "Bash",
  "tool_input": { "command": "..." },
  "reason": "..."
}
```

- `--permission-mode auto` の分類器が `deny` と判断した時のみ発火。retry したいかどうかを外から制御できる

---

### サブエージェント・タスク系

#### `SubagentStart` / `SubagentStop`

```json
{
  "hook_event_name": "SubagentStart",
  "agent_id": "agent_...",
  "agent_type": "general-purpose"
}
```

```json
{
  "hook_event_name": "SubagentStop",
  "agent_id": "agent_...",
  "agent_type": "general-purpose",
  "agent_transcript_path": "~/.claude/projects/.../<agent_id>.jsonl",
  "last_assistant_message": "...",
  "stop_hook_active": true
}
```

- サブエージェントの **独立した transcript_path が取れる**。親セッションの JSONL とは別ファイルに吐かれている
- `stop_hook_active` は Stop hook による無限ループ抑止フラグ

#### `TaskCreated` / `TaskCompleted`

```json
{
  "hook_event_name": "TaskCreated",
  "task_id": "...",
  "task_subject": "...",
  "task_description": "...",
  "teammate_name": "...",
  "team_name": "..."
}
```

- Claude Code 内蔵の `TaskCreate` ツールで task が作られた/完了した瞬間。外部 issue tracker に同期するときに使える
- exit 2 で task 作成自体をブロックできる

#### `TeammateIdle`

```json
{
  "hook_event_name": "TeammateIdle",
  "teammate_name": "...",
  "team_name": "...",
  "idle_reason": "..."
}
```

- Claude Code 公式の agent team 機能で teammate が idle になった時に発火。独自に team 機能を作る場合は参考程度

---

### 設定・ファイル系

#### `InstructionsLoaded`

```json
{
  "hook_event_name": "InstructionsLoaded",
  "file_path": "CLAUDE.md",
  "memory_type": "User|Project|Local|Managed",
  "load_reason": "session_start|nested_traversal|path_glob_match|include|compact",
  "globs": ["**/*.ts"],
  "trigger_file_path": "...",
  "parent_file_path": "..."
}
```

- どの CLAUDE.md / `.claude/rules/*.md` が読み込まれたかが分かる。**`compact` 時にも発火する**点は地味に重要（compact 後の文脈で読み直しが走る）

#### `ConfigChange`

```json
{
  "hook_event_name": "ConfigChange",
  "config_source": "user_settings|project_settings|local_settings|policy_settings|skills",
  "file_path": "..."
}
```

- 実行中に settings.json を書き換えたタイミングで発火。**exit 2 で変更をブロック可能**

#### `CwdChanged`

```json
{
  "hook_event_name": "CwdChanged",
  "previous_cwd": "...",
  "new_cwd": "..."
}
```

- cwd が変わった時だけ通知。resume の cwd 問題（後述）を監視したいとき便利

#### `FileChanged`

```json
{
  "hook_event_name": "FileChanged",
  "file_path": "...",
  "change_type": "created|modified|deleted"
}
```

- matcher はファイル名の正規表現。`".env$"` で .env の変更だけ拾う、みたいな使い方

#### `WorktreeCreate` / `WorktreeRemove`

```json
{
  "hook_event_name": "WorktreeCreate",
  "worktree_name": "...",
  "base_branch": "...",
  "new_branch": "...",
  "repo_path": "..."
}
```

- worktree 作成時に `.envrc` を流し込む、`.claude/settings.local.json` をコピーする、等のセットアップに最適

---

### 圧縮・通知・MCP 系

#### `PreCompact` / `PostCompact`

```json
{
  "hook_event_name": "PreCompact",
  "compact_trigger": "manual|auto"
}
```

- matcher は `manual` / `auto`。手動 `/compact` と自動 compaction の両方フック可能
- **compact は破壊的操作**なので、`PreCompact` で `transcript_path` を別ファイルにコピーしておくのが実用 TIPS（別節の「`/compact` 前の会話を読み返す」と合わせ技）

#### `Notification`

```json
{
  "hook_event_name": "Notification",
  "message": "...",
  "title": "...",
  "notification_type": "permission_prompt|idle_prompt|auth_success|elicitation_dialog"
}
```

- `permission_prompt` を拾えば、**権限ダイアログが出たことを外から即検知**できる。Slack や Discord に飛ばしてリモートで承認指示、みたいな基盤が作れる
- `idle_prompt` もあるが、Stop hook のほうがほぼ同じ情報を早く取れるので出番は少ない

#### `Elicitation` / `ElicitationResult`

```json
{
  "hook_event_name": "Elicitation",
  "mcp_server_name": "...",
  "tool_name": "...",
  "elicitation_fields": [{ "name": "...", "type": "...", "description": "..." }]
}
```

- MCP サーバが**ユーザ入力を要求してきた**時に発火。CI で対話不能な環境だと exit 2 で拒否すればすぐフェイルできる
- `ElicitationResult` は応答後の値も含むので、そのままログに残すと監査証跡になる

---

#### `--settings` でロール別に hook を差し替える

hook 定義は `.claude/settings.json` 以外に、Claude Code の **`--settings` オプションで起動ごとに任意の JSON を差し替え可能** です。cmux-team では Conductor / Agent / Master で別々の settings JSON を生成し、それぞれ違う hook を仕込んでいます。ロールごとに通知内容や matcher を変えたい場合はこの方式が一番綺麗です。

### 子プロセスの生存確認は `process.kill(pid, 0)` が最安

Claude がクラッシュしたことを検出したい場合、`process.kill(pid, 0)` が最軽量な手段です。UNIX のシグナル 0 は「プロセス存在テストのみ」を行い、副作用がありません。

```ts
setInterval(() => {
  try {
    process.kill(claudePid, 0);
  } catch {
    // ESRCH = プロセスが存在しない
    handleCrash();
  }
}, 1000);
```

`ps` や `kill -0` を shell out するより軽く、Node/Bun の try/catch で完結します。`SessionStart` hook で `$PPID` を親に通知しておけば PID を取得できます。

### デーモンの自己再起動は exit code 42 で足りる

systemd や pm2 を持ち出さなくても、CLI ラッパーで `execFileSync` をループさせて特定の exit code だけ再起動する方法で十分です。

```js
// bin/cmux-team.js
let restarts = 0;
while (restarts < MAX_RESTARTS) {
  try {
    execFileSync('node', ['daemon.js'], { stdio: 'inherit' });
    break; // 正常終了
  } catch (e) {
    if (e.status === 42) {
      restarts++;
      continue; // 設定変更 → 再起動
    }
    throw e;
  }
}
```

プロセス側は設定変更を反映したいときに `process.exit(42)` するだけです。再起動回数に上限を設けておけば暴走も防げます。

## worktree と環境変数

### worktree で環境変数が消える問題

`git worktree add` で作ったディレクトリには tracked files しか含まれません。`.envrc` が欠落して**環境変数がまるごと消えます**。

解決策は worktree 内に `source_up` の 1 行だけ書いた `.envrc` を生成することです。

```bash
echo "source_up" > /path/to/worktree/.envrc
direnv allow /path/to/worktree
```

direnv が親ディレクトリを遡って本物の `.envrc` を読んでくれます。また `.claude/settings.local.json` も untracked なので、worktree に明示的にコピーする必要があります。

## ロール別システムプロンプト

長大なロール定義プロンプトを Conductor に渡すとき、`--append-system-prompt` だと文字列をシェル引数に展開することになり、エスケープ地獄になります。

**`--append-system-prompt-file` でファイルパスを渡す**と解決します。

```ts
const promptFile = await writeTemp(rolePrompt);
execFile('claude', [
  '--session-id', sessionId,
  '--model', 'claude-opus-4-5',
  '--settings', settingsFile,
  '--append-system-prompt-file', promptFile,
]);
```

`--settings` でhookを差し替えつつ `--append-system-prompt-file` でロール定義を差し替えると、1 バイナリで「Researcher 用 Claude」「Implementer 用 Claude」を使い分けられます。

## おわりに

Claude Code で AI オーケストレーション基盤を自作すると、ドキュメントの外にある挙動に頻繁に出くわします。この記事で挙げたものはほとんど実装して初めて気づいたものです。

同じようなものを作ろうとしている人の参考になれば幸いです。実装の詳細は [cmux-team リポジトリ](https://github.com/hummer98/cmux-team) のソースを見てみてください。

## リンク

- [cmux-team リポジトリ](https://github.com/hummer98/cmux-team)
- [cmux-team v3 アーキテクチャ記事](https://zenn.dev/hummer/articles/cmux-team-multi-agent)
- [cmux エコシステム記事](https://zenn.dev/hummer/articles/cmux-ecosystem-claude-code)
