# STORYLINE 統合導入ガイド

## 採用構成

Notionを企画・素材依頼・承認・生成ジョブの正本DBとして使用し、n8nをオーケストレーター、GitHub ActionsをRemotionの並列レンダリング、Kaggle GPUをLivePortrait / SadTalkerのバッチ実行環境として使います。Oracle Cloudは使用しません。

| 役割 | 実行場所 |
|---|---|
| 管理画面・既存Node.js API | 現在のNode.js環境 |
| ワークフロー・Webhook | n8n |
| 企画・素材・承認の正本 | Notion Database |
| コードベース動画生成 | GitHub Actions + Remotion |
| GPUキャラクター動画 | Kaggle Notebook + LivePortrait / SadTalker |

## 初期設定

1. `.env.example` を `.env` にコピーする。
2. Notionでデータベースを作成し、Integrationを招待する。
3. データベースに次のプロパティを作成する。
   - `Name`: Title
   - `Channel`: Rich text
   - `Stage`: Select
   - `Status`: Select
   - `UpdatedAt`: Date
   - `StorylineId`: Rich text
4. `NOTION_TOKEN` と `NOTION_DATABASE_ID` を設定する。
5. `n8n/storyline-workflow.json` をn8nへImportし、Webhook URLと秘密文字列を設定する。
6. `.github/workflows/render-remotion.yml` をGitHubリポジトリへ配置し、Actionsを有効化する。
7. `remotion/` をそのリポジトリに含め、`npm ci`でRemotion依存をインストールする。
8. Kaggle Notebookへ `kaggle/storyline_gpu_worker.py` を配置し、GPU Acceleratorを有効にする。LivePortraitまたはSadTalkerのモデルとリポジトリはKaggle Datasetとして準備する。

## セキュリティ

APIキーをコード、Notion、GitHub Actionsのログ、Kaggle Notebookへ直接書き込まない。GitHubではRepository Secrets、n8nではCredentialsまたは環境変数、KaggleではSecretsを使用する。Webhookには `N8N_WEBHOOK_SECRET` を必ず設定する。

## ジョブ契約

n8nへ送るジョブは次の形にする。

```json
{
  "secret": "設定した秘密文字列",
  "jobId": "render_123",
  "type": "render",
  "projectId": "project_123",
  "payload": {
    "title": "動画タイトル",
    "duration": 300,
    "scenes": []
  }
}
```

GitHub Actionsへは `repository_dispatch` の `storyline-render` を使い、`client_payload.job_id` と `client_payload.project` を渡す。Kaggleは常時サーバーではないため、n8nからNotebook実行またはDataset更新をトリガーするバッチ実行環境として扱う。

## Remotionについて

Remotionは現在のFFmpegレンダラーを直ちに削除せず、まず並列レンダリング用の別経路として導入する。成功したジョブだけを完成動画として採用し、失敗時は既存FFmpeg経路へ戻せるようにする。

## LivePortrait / SadTalkerについて

LivePortraitとSadTalkerはGPU依存のPython処理なので、Node.js APIと同一プロセスへ組み込まない。Kaggle側で画像・音声からキャラクター動画を作り、成果物URLまたは成果物Datasetをn8n経由で取り込む。
