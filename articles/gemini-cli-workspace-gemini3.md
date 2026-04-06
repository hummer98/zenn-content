---
title: "Google Workspace契約者がGemini CLIでGemini 3.1 Proを使うまでの地獄"
emoji: "🔑"
type: "tech"
topics: ["gemini", "geminicli", "googleworkspace", "gcp", "ai"]
published: true
slug: "gemini-cli-workspace-gemini3"
---

## はじめに

「Gemini CLI で Gemini 3.1 Pro が使えるようになった！」という[公式ブログ](https://cloud.google.com/blog/products/ai-machine-learning/gemini-3-1-pro-on-gemini-cli-gemini-enterprise-and-vertex-ai)を見て、早速試してみました。

```
 > /model
 Auto (Gemini 2.5)
```

**Gemini 2.5 しか出てこない。**

原因を調べていくと、混乱ポイントが3層に重なっていました。この記事ではその全体像を整理し、実際に Gemini 3.1 Pro を使えるようになるまでの手順をまとめます。

## 混乱ポイント0：「Gemini 3.1 Pro が使える」という宣伝の実態

そもそも、Workspace のプラン説明で「Gemini 3.1 Pro が使える」と明記されているかを確認してみました。

- **[Workspace 料金ページ](https://workspace.google.co.jp/pricing?hl=ja)**: モデル名の記載は**一切なし**。Business Plus の記述は「Gemini アプリで AI と会話、幅広いモデルと機能を利用可能」という曖昧な表現のみ。
- **AI Ultra Access アドオンページ**: "Gemini 3 Pro" の記述はあるが、これは**別途購入が必要なアドオン**の説明。

つまり「Gemini 3.1 Pro が使える」は Workspace プランの標準的な謳い文句ではなく、Gemini アプリ（gemini.google.com）文脈の話がそのまま流れてきているようです。開発ツールである Gemini CLI での話とは、また別です。

## 混乱ポイント1：Gemini Advanced と Code Assist は別製品

「Workspace に Gemini Advanced が含まれるようになったから最新モデルが使えるはず」と思っていたのですが、これが最初の落とし穴でした。

2025年1月、Google は **Gemini for Google Workspace アドオン**の販売を終了し、Gemini Advanced 相当の機能を Business Plus / Enterprise の標準プランに組み込みました[^1]。ただしこれは **Workspace アプリ内の AI 機能**の話です。

| 機能 | Business Plus に含まれるか |
|------|--------------------------|
| Gmail / Docs / Sheets の AI 文章生成 | ✅ 含まれる |
| Gemini.google.com（Gemini Advanced） | ✅ 含まれる |
| NotebookLM Plus | ✅ 含まれる |
| **Gemini CLI（コーディング支援）** | **❌ 含まれない** |
| **Gemini Code Assist Standard** | **❌ 別途購入が必要（公式には）** |

**Gemini Code Assist** は **Gemini for Google Cloud** ポートフォリオの製品であり、Workspace プランとは完全に別の契約です。

Code Assist ライセンスの有無は [Google Admin コンソール](https://admin.google.com) → **ユーザー** → **ライセンス** タブで確認できます。「Gemini Code Assist Standard」が表示されていなければ未割り当てです。

[^1]: [Gemini AI features now included in Google Workspace subscriptions](https://knowledge.workspace.google.com/admin/gemini/gemini-ai-features-now-included-in-google-workspace-subscriptions)、[Google Workspace の価格改定と Gemini の標準搭載について](https://www.zunda.co.jp/blog/google-workspace-pricing-update)

## 混乱ポイント2：デフォルトチャネルでは Gemini 3.1 Pro は使えない

Code Assist Standard ライセンスがあっても、デフォルトでは Gemini 3.1 Pro は使えません。

Workspace の Code Assist には**リリースチャネル**という仕組みがあり、デフォルトは `STABLE`（GA モデルのみ）です。Gemini 3.1 Pro はまだ Preview 扱いのため、チャネルを `EXPERIMENTAL` に変更しない限り選択肢に現れません。

そしてこのチャネル変更は、**`gcloud` コマンドでは直接できません**。REST API を叩く必要があります。

## 混乱ポイント3：ドキュメントが分散している

この一連の設定をまとめた公式ドキュメントは存在しません。必要な情報が以下のように分散しています。

| 内容 | ドキュメント |
|------|------------|
| Code Assist セットアップ全般 | [Set up Gemini Code Assist Standard and Enterprise](https://docs.cloud.google.com/gemini/docs/codeassist/set-up-gemini) |
| Gemini CLI の認証方法 | [Gemini CLI (Google Cloud)](https://docs.cloud.google.com/gemini/docs/codeassist/gemini-cli) |
| Gemini 3 の対応状況・プラン別可否 | [Gemini 3 in Gemini Code Assist](https://developers.google.com/gemini-code-assist/docs/gemini-3) |
| Preview チャネルの有効化（REST API） | [Configure release channels](https://docs.cloud.google.com/gemini/docs/codeassist/configure-release-channels) |

---

## 設定手順

### 1. GCP: Cloud AI Companion API を有効化

```bash
gcloud services enable cloudaicompanion.googleapis.com \
  --project YOUR_PROJECT_ID \
  --account ADMIN@example.com
```

確認:

```bash
gcloud services list --enabled --project YOUR_PROJECT_ID \
  --filter="name:cloudaicompanion"
# NAME                             TITLE
# cloudaicompanion.googleapis.com  Gemini for Google Cloud API
```

### 2. GCP: IAM ロールを付与

```bash
# 利用ユーザーに付与
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:USER@example.com" \
  --role="roles/cloudaicompanion.user"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:USER@example.com" \
  --role="roles/serviceusage.serviceUsageConsumer"

# チャネル設定を行う管理者にも必要
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:ADMIN@example.com" \
  --role="roles/cloudaicompanion.settingsAdmin"
```

### 3. REST API: Preview チャネルを有効化

`gcloud` コマンドにこの機能はないため、REST API を直接叩きます。

```bash
TOKEN=$(gcloud auth print-access-token --account ADMIN@example.com)

# リリースチャネル設定を作成
curl \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"release_channel": "EXPERIMENTAL"}' \
  -X POST \
  "https://cloudaicompanion.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/global/releaseChannelSettings?release_channel_setting_id=rc1"
```

成功すると以下が返ります:

```json
{
  "name": "projects/YOUR_PROJECT_ID/locations/global/releaseChannelSettings/rc1",
  "releaseChannel": "EXPERIMENTAL",
  "createTime": "2026-04-06T..."
}
```

設定を作るだけでは足りません。プロジェクトへの**バインディング**も作成します。

```bash
curl \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"target": "projects/YOUR_PROJECT_ID", "product": "GEMINI_CODE_ASSIST"}' \
  -X POST \
  "https://cloudaicompanion.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/global/releaseChannelSettings/rc1/settingBindings?setting_binding_id=rc_binding1"
```

バインディング作成は非同期です。完了確認:

```bash
curl \
  -H "Authorization: Bearer $TOKEN" \
  "https://cloudaicompanion.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/global/operations/OPERATION_ID" \
  | jq '{done, error}'
# { "done": true, "error": null }
```

### 4. Gemini CLI のインストールと認証

```bash
npm install -g @google/gemini-cli

# Workspace ユーザーは GCP プロジェクトの設定が必須
export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"

gemini
```

初回起動時に「Sign in with Google」を選択し、ブラウザで Workspace アカウントにサインインします。

---

## 結果

設定後に `/model` を確認すると：

```
 Select Model

 ● 1. Auto (Gemini 3)
      Let Gemini CLI decide the best model for the task: gemini-3.1-pro, gemini-3-flash
   2. Auto (Gemini 2.5)
      Let Gemini CLI decide the best model for the task: gemini-2.5-pro, gemini-2.5-flash
   3. Manual
      Manually select a model
```

設定前は `Auto (Gemini 2.5)` しか選べなかった状態から、`Auto (Gemini 3)` が選択肢に現れました。

## おまけ：Code Assist Standard ライセンスなしでも動いた

実際には、今回の設定を **Code Assist Standard ライセンスなし**（Workspace アカウントで OAuth ログインした無料枠）で行いましたが、Gemini 3.1 Pro が使えるようになりました。

```
Plan: Gemini Code Assist  /upgrade   ← 有料ライセンスなしの状態
...
Auto (Gemini 3)                      ← でも Gemini 3 が使える
```

公式ドキュメントでは「Code Assist Standard ライセンスが前提」とされていますが、GCP プロジェクトに EXPERIMENTAL チャネルを設定するだけで解放される模様です。レート制限や利用上限の詳細は不明なので自己責任で。

## おわりに

「新しいモデル出た！試してみよう」→「Gemini 2.5 しか出てこない」→「Workspace の Gemini Advanced とは別製品だった」→「ライセンスなしでも、リリースチャネルを REST API で叩けば動いた」という旅でした。

Gemini 3.1 Pro が正式 GA になり、`gcloud` コマンド一発で設定できるようになる頃には、この記事が不要になることを願っています。
