# Azure SDLC Pipeline Template

Azure DevOps SDLC Pipeline YAML Template — 一套基於容器化架構的 CI/CD 管線範本，整合 **AI Code Review** 功能，可在 Pull Request 階段自動執行程式碼審查並將報告回貼至 PR 留言板。

---

## 📂 檔案總覽

| 檔案                        | 用途                                                                    |
| --------------------------- | ----------------------------------------------------------------------- |
| `sdlc-master-template.yaml` | **主範本** — 定義 Job、啟動容器、編排步驟、清理容器                     |
| `code-review-template.yaml` | **Code Review 步驟範本** — 被主範本引用，負責 AI Code Review 的完整流程 |

---

## 🏗️ 架構概述

```
呼叫端 Pipeline
  └─ sdlc-master-template.yaml  (主範本)
       ├─ 1. 啟動 Podman 容器
       ├─ 2. code-review-template.yaml  (Code Review 步驟)
       │    ├─ Checkout 原始碼 & 共用 Gemini 檔案
       │    ├─ 載入 Skill / Policy 設定
       │    ├─ 準備 Git 環境 (偵測 main/master)
       │    ├─ 執行 AI Code Review (Gemini CLI)
       │    └─ 發布報告至 PR 留言
       ├─ 3. 呼叫端自定義步驟 (buildSteps)
       └─ 4. 清理容器
```

---

## 📄 sdlc-master-template.yaml

> **角色**：主要入口範本，負責容器生命週期管理與步驟編排。

### Parameters

| 參數名稱     | 類型       | 預設值 | 說明                         |
| ------------ | ---------- | ------ | ---------------------------- |
| `buildSteps` | `stepList` | `[]`   | 呼叫端可傳入的自定義建置步驟 |

### Resources

| 代號          | 類型  | 來源                   | 說明                                              |
| ------------- | ----- | ---------------------- | ------------------------------------------------- |
| `gemini-file` | `git` | `{ProjectName}/GEMINI` | 存放共用 Gemini CLI Skills / Policy / 憑證的 Repo |

### 執行流程

| #   | 步驟                | 說明                                                                                                                                                |
| --- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Start container** | 使用 `podman run` 啟動 `ubi8/python-311` 容器，掛載 Agent 工作目錄 (`/workspace`) 及工具目錄 (`/azp/externals`)，採用 `--network=host` 共用主機網路 |
| 2   | **Code Review**     | 引用 `code-review-template.yaml`，於容器內執行 AI Code Review                                                                                       |
| 3   | **自定義步驟**      | 透過 `${{ parameters.buildSteps }}` 插入呼叫端傳入的建置 / 測試等步驟                                                                               |
| 4   | **Clean up**        | 無論前面步驟成功或失敗（`condition: always()`），強制移除容器                                                                                       |

### 容器掛載對照

| 主機路徑                  | 容器路徑         | 用途              |
| ------------------------- | ---------------- | ----------------- |
| `$(Agent.BuildDirectory)` | `/workspace`     | 原始碼與工作目錄  |
| `/workspace/tools`        | `/azp/externals` | Gemini CLI 等工具 |

---

## 📄 code-review-template.yaml

> **角色**：AI Code Review 的完整執行流程，由 `sdlc-master-template.yaml` 以 `template` 方式引用。

### 執行流程

| #   | displayName                        | 條件             | 說明                                                                                                    |
| --- | ---------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------- |
| 1   | **Checkout**                       | —                | Checkout 當前 Repo（啟用 `persistCredentials`）及 `gemini-file` Repo                                    |
| 2   | **環境測試與歡迎訊息**             | —                | 在容器內列印觸發資訊與 Agent 名稱，驗證環境可用性                                                       |
| 3   | **載入 Gemini CLI、Skill、Policy** | —                | 將 `gemini-file` Repo 的內容複製到 `/tmp/gemini_file/.gemini`，供 Gemini CLI 使用                       |
| 4   | **準備 Git 環境**                  | `PullRequest` 時 | 設定 Git safe directory，自動偵測遠端主分支（`main` 或 `master`），並 `fetch` 該分支作為 diff 比較基準  |
| 5   | **設定權限並執行 AI Code Review**  | —                | 設定 Node / Gemini 執行權限，配置 Vertex AI 環境變數，呼叫 `gemini -y -p` 執行 code-review-skill        |
| 6   | **發布報告至 PR 留言**             | `PullRequest` 時 | 讀取 `code-review/references/report/` 下最新 Markdown 報告，透過 Azure DevOps REST API 發布至 PR Thread |

### 環境變數一覽

| 變數名稱                         | 值 / 來源                                                                            | 說明                                      |
| -------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------- |
| `GOOGLE_GENAI_USE_VERTEXAI`      | `true`                                                                               | 使用 Vertex AI 後端                       |
| `GOOGLE_CLOUD_PROJECT`           | Pipeline 變數                                                                        | GCP 專案 ID                               |
| `GOOGLE_CLOUD_LOCATION`          | `global`                                                                             | Vertex AI 區域                            |
| `GOOGLE_APPLICATION_CREDENTIALS` | `/tmp/gemini_file/.gemini/cert/{VertexAI-Service-Account-Private-Key-FileName}.json` | 服務帳號憑證                              |
| `GEMINI_CLI_HOME`                | `/tmp/gemini_file/`                                                                  | Gemini CLI 設定目錄（含 Skills / Policy） |
| `TZ`                             | `Asia/Taipei`                                                                        | 容器時區                                  |

### PR 留言發布流程

1. 在容器外（Host Agent）產生 `send_pr_comment.sh` 腳本
2. 透過 `podman exec` 注入 Azure DevOps 系統變數後執行腳本
3. 腳本流程：
   - 從 `/workspace/<repo>/code-review/references/report/` 找到最新 `.md` 報告
   - 使用 `jq` 將報告包裝為 Azure DevOps PR Thread API 所需的 JSON Payload
   - 呼叫 `curl` 發送 POST 請求到 `_apis/git/repositories/{repoId}/pullRequests/{prId}/threads`
   - 回傳 HTTP 狀態碼判斷成功與否

### 使用到的 Azure DevOps 系統變數

| 變數                               | 用途                              |
| ---------------------------------- | --------------------------------- |
| `System.CollectionUri`             | ADO 組織 URL                      |
| `System.TeamProjectId`             | 專案 ID                           |
| `Build.Repository.ID`              | Repo ID                           |
| `System.PullRequest.PullRequestId` | PR 編號                           |
| `System.AccessToken`               | OAuth Token（需在 Pipeline 授權） |

---

## 🚀 使用方式

在呼叫端的 `azure-pipelines.yml` 中引用此共用範本：

```yaml
resources:
  repositories:
    - repository: templates_repo
      type: git
      name: {ProjectName}/azure-sdlc-pipeline-template

extends:
  template: sdlc-master-template.yaml@templates_repo
  parameters:
    buildSteps:
      - script: echo "這是專案專有的構建動作"
        displayName: 'Project Specific Build'
```

---

## ⚠️ 前置需求

- **Self-hosted Agent Pool**：需設定已建立的 Agent Pool Name
- **Podman**：Agent 主機需安裝 Podman
- **Gemini CLI**：已預先部署至 `/workspace/tools` 目錄
- **GCP Vertex AI 憑證**：服務帳號 JSON 金鑰需存放於 `gemini-file` Repo 的 `cert/` 目錄
- **Pipeline 變數**：需設定 `GOOGLE_CLOUD_PROJECT` 變數
- **PR Build 權限**：需開啟 `System.AccessToken` 的 PR 留言寫入權限
