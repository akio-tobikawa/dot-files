# 外部スケジューラから webhook で日次セレクションを起動する手順

GitHub Actions の `workflow_dispatch` REST API を webhook エンドポイントとして使用します。
Notion / Google Calendar / Zapier / Make / 任意の cron サービス、すべてからこの一つのエンドポイントを叩くだけで起動できます。

---

## 1. 事前準備（一度だけ）

### 1.1 リポジトリに GitHub Actions Secret を登録

GitHub の Web UI でリポジトリ → **Settings → Secrets and variables → Actions → New repository secret**：

| Name | Value |
|------|-------|
| `ANTHROPIC_API_KEY` | Anthropic コンソールで発行した API キー（`sk-ant-...`） |

> 本ワークフローは Claude Code CLI を Actions ランナーにインストールして実行するため、Anthropic API キーが必須。

### 1.2 GitHub Personal Access Token (PAT) を発行

外部スケジューラから REST API を叩くために PAT が必要です。

1. GitHub → 右上アバター → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**
2. Repository access: `akio-tobikawa/dot-files` のみ
3. Permissions → Repository permissions：
   - **Actions: Read and write**
   - **Contents: Read** （`workflow_dispatch` 自体には不要だが、トリガー後の動作確認に便利）
4. 発行された `github_pat_...` を安全に保管。

> 注意：classic PAT を使う場合は `repo` スコープ＋ `workflow` スコープが必要。fine-grained が推奨。

---

## 2. Webhook エンドポイント（共通）

```
POST https://api.github.com/repos/akio-tobikawa/dot-files/actions/workflows/daily-antimicrobial-papers.yml/dispatches
```

ヘッダ：
```
Authorization: Bearer <GITHUB_PAT>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
Content-Type: application/json
```

ボディ：
```json
{
  "ref": "claude/medical-papers-selection-A0m2x",
  "inputs": {
    "reason": "scheduled by Notion automation"
  }
}
```

成功時：HTTP `204 No Content`（レスポンスボディなし）。

### 動作確認用 curl

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_PAT" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/akio-tobikawa/dot-files/actions/workflows/daily-antimicrobial-papers.yml/dispatches \
  -d '{"ref":"claude/medical-papers-selection-A0m2x","inputs":{"reason":"manual test"}}'
```

---

## 3. Notion から起動する

Notion の **Database Automations** または **Button** から HTTP リクエストを送信する方法は二通りあります。

### 3.1 Notion Automations（Business / Enterprise プラン）

Notion 内蔵の Automation で「**Send webhook**」アクションを使い、上記エンドポイントを設定。
- URL: 上記エンドポイント
- Method: `POST`
- Headers:
  - `Authorization: Bearer <GITHUB_PAT>`
  - `Accept: application/vnd.github+json`
  - `Content-Type: application/json`
- Body: 上記 JSON

トリガーには「Every day at 07:00 JST」など。

### 3.2 Notion Button → サードパーティ（Free / Plus でも可）

Notion の Button アクションで「**Open link**」または、N8n / Make / Zapier 経由でフックする：
- Button → Webhook URL（中継サービス側）
- 中継サービス → GitHub REST API

---

## 4. Google Calendar から起動する

Google Apps Script で日次イベントをトリガーに REST API を呼び出します。

1. [script.google.com](https://script.google.com) → 新規プロジェクト。
2. 以下のコードを貼り付け：

```javascript
function triggerDailyAntimicrobialPapers() {
  const url = 'https://api.github.com/repos/akio-tobikawa/dot-files/actions/workflows/daily-antimicrobial-papers.yml/dispatches';
  const pat = PropertiesService.getScriptProperties().getProperty('GITHUB_PAT');
  const payload = {
    ref: 'claude/medical-papers-selection-A0m2x',
    inputs: { reason: 'Google Apps Script daily trigger' }
  };
  UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: {
      Authorization: 'Bearer ' + pat,
      Accept: 'application/vnd.github+json',
      'X-GitHub-Api-Version': '2022-11-28'
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });
}
```

3. **Project Settings → Script Properties** に `GITHUB_PAT = <PAT>` を追加。
4. **Triggers → Add Trigger** で `triggerDailyAntimicrobialPapers` を「Time-driven / Day timer / 07:00–08:00」に設定。

Google Calendar イベントに連動させたい場合は、Calendar API トリガーまたは「特定タイトルのイベント開始時」を Apps Script でハンドリング可能。

---

## 5. Zapier / Make / n8n から起動する

すべて「**Webhooks → POST**」モジュールに上記エンドポイント・ヘッダ・ボディを設定するだけ。
スケジューラ部分（Schedule by Zapier 等）と組み合わせて日次トリガーに。

---

## 6. ワークフロー本体の挙動

- 起動後、`claude/medical-papers-selection-A0m2x` ブランチをチェックアウト
- Claude Code CLI を npm でインストール
- PubMed MCP を `uvx` でセットアップ
- `prompts/daily-antimicrobial-selection.md` を読み込み、Claude に渡して実行
- Claude が PubMed 検索 → 既報管理照合 → 抗菌薬.md 更新 → 既報管理.md 追記
- 変更があれば自動コミット → 同ブランチへ push

`schedule:` で 22:00 UTC（07:00 JST）にも自動起動するので、外部スケジューラがダウンしてもフェイルセーフとして動きます。外部スケジューラのみで運用したい場合は、ワークフロー YAML から `schedule:` ブロックを削除してください。

---

## 7. 失敗時のリカバリ

- ワークフロー結果は **Actions タブ** で確認。
- 失敗した日は **手動 re-run** または手元での `claude` 実行で代替可。
- API キーローテーションは `ANTHROPIC_API_KEY` シークレットを更新するだけ。
- PAT 期限切れは外部スケジューラ側で更新。

---

## 8. セキュリティ留意点

- PAT は **fine-grained** で、対象リポジトリ・Actions write のみに権限を絞る。
- ANTHROPIC_API_KEY は GitHub Secrets（暗号化）に保管し、ログには出さない。
- Notion/Google Apps Script 側でも、Script Properties / 暗号化フィールドに格納し、ワークシート等の平文に書かない。
- 外部 webhook サービスを経由する場合、PAT が中継先のログに残らないかを必ず確認。
