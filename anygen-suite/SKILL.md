---
name: anygen
version: 2.0.0
description: "AnyGen: AI-powered content creation suite. Create slides/PPT, documents, diagrams, websites, data visualizations, research reports, storybooks, financial analysis, and images. Supports: pitch decks, keynotes, technical docs, PRDs, white papers, architecture diagrams, flowcharts, mind maps, org charts, ER diagrams, sequence diagrams, UML, landing pages, CSV analysis, earnings research, posters, banners, comics, and more. Also trigger when: 做PPT, 写文档, 画流程图, 做网站, 分析数据, 帮我调研, 做绘本, 分析财报, 生成图片, 做海报, 思维导图, 做个架构图, 季度汇报, 竞品调研, 技术方案, 建个落地页, 做个估值, 画个故事."
metadata:
  requires:
    bins: ["anygen"]
    env: ["ANYGEN_API_KEY"]
  install:
    - id: node
      kind: node
      package: "@anygen/cli"
      bins: ["anygen"]
  cliHelp: "anygen --help"
---

# AnyGen — Content Generation Workflow

## Authentication

```bash
# Web login (recommended for agent usage)
anygen auth login --no-wait

# Direct API key (no browser needed)
anygen auth login --api-key sk-xxx

# Environment variable
export ANYGEN_API_KEY=sk-xxx
```

When prompting the user for an API key, MUST use Markdown link: [Get your AnyGen API Key](https://www.anygen.io/home?auto_create_openclaw_key=1)

## Rules

**Follow these rules exactly.**

- Schema: run `anygen schema <resource.method>` to check required params and response if needed.
- Long-running: `--wait` commands will block, MUST use `sessions_spawn` to run in the background.
- Sending files on Feishu/Lark: Do not use the message tool to send files. It corrupts non-ASCII filenames into `%XX` garbage. Strictly follow the curl process in "Sending files".
- **Never** output API keys or auth tokens directly.
- **Always** confirm with user before uploading files or creating tasks.
- Use natural language instead of exposing task_id, file_token, or CLI syntax to the user.
- Always return links using Markdown format: `[text](url)`.

## Steps

1. **Discover operations metadata**:
   `anygen task operations`
   Do not guess operation types. Always run to get supported operations and their estimated time and thumbnail support.

2. **Upload reference files** (skip if no reference files):
   `anygen file upload --data '{"file":"./data.csv"}'`
   → Save `file_token` for step 4. Tell user the file was uploaded.

3. **Gather requirements** (skip if requirements are already clear):
   `anygen task prepare --data '{"operation":"slide","messages":[{"role":"user","content":"Make a Q4 report PPT"}]}'`
   Present `reply` to user, collect their answer, then call again with `prepare_session_id` and updated `messages`:
   `anygen task prepare --data '{"operation":"slide","prepare_session_id":"<id>","messages":[...previous messages...,{"role":"user","content":"user's answer"}]}'`
   Repeat until `status=ready`.
   → When ready, show `suggested_task_params.prompt` as outline, confirm with user, then use it as `prompt` in step 4.

4. **Create task**:
   `anygen task create --data '{"operation":"slide","prompt":"...","file_tokens":["<file_token>"]}'`
   → Tell user the task is created, share `task_url` and estimated time (from step 1).

5. **Wait for completion** (long-running, must run in background via `sessions_spawn`):
   `anygen task get --params '{"task_id":"<id>"}' --wait`

6. **Deliver** (after step 5 completes, check the result):
   - **No files** (`output.files` empty): show `message` to user if present.
   - **Has files + has thumbnail** (`has_thumbnail` from step 1):
     `anygen task +download --task-id <id> --thumbnail`
     → Send thumbnail image with `task_url` as preview. Do not download files yet — wait for user to request download or modifications (→ step 7).
   - **Has files + no thumbnail**:
     `anygen task +download --task-id <id>`
     → Send files to user (see "Sending files" below).

7. **Modify** (on user request):
   `anygen task message send --params '{"task_id":"<id>"}' --data '{"content":"..."}'`
   Then wait for result (long-running, must run in background via `sessions_spawn`):
   `anygen task message list --params '{"task_id":"<id>"}' --wait`
   → Repeat from step 5 to re-export and deliver. All modifications reuse the same task.

## Sending files

When user requests file download, or when delivering files from step 6:
`anygen task +download --task-id <id>`
To download specific files: `anygen task +download --task-id <id> --file report.pptx`

**Feishu/Lark** (message tool corrupts non-ASCII filenames, use curl instead):
1. Get credentials: read `app_id` and `app_secret` from the config file (e.g. `cat ~/.openclaw/openclaw.json | jq '.channels.feishu'` instead of `openclaw config get`). Make sure to use the credentials matching the current account.
2. Get token: `curl -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' -H 'Content-Type: application/json' -d '{"app_id":"<app_id>","app_secret":"<app_secret>"}'`
3. Upload + Send per file type:
   - **Images** (thumbnail, png, jpg, etc.):
     Upload: `curl -X POST 'https://open.feishu.cn/open-apis/im/v1/images' -H 'Authorization: Bearer <tenant_access_token>' -F 'image_type=message' -F 'image=@./preview.png'`
     Send: `curl -X POST 'https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id' -H 'Authorization: Bearer <tenant_access_token>' -H 'Content-Type: application/json' -d '{"receive_id":"<chat_id>","msg_type":"image","content":"{\"image_key\":\"<image_key>\"}"}'`
   - **Documents** (pptx/docx/pdf, etc.):
     Upload: `curl -X POST 'https://open.feishu.cn/open-apis/im/v1/files' -H 'Authorization: Bearer <tenant_access_token>' -F 'file_type=ppt' -F 'file=@./output.pptx' -F 'file_name=output.pptx'`
     `file_type` values: `opus` (audio), `mp4` (video), `pdf`, `doc`, `xls`, `ppt`, `stream` (other).
     Send: `curl -X POST 'https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id' -H 'Authorization: Bearer <tenant_access_token>' -H 'Content-Type: application/json' -d '{"receive_id":"<chat_id>","msg_type":"file","content":"{\"file_key\":\"<file_key>\"}"}'`

**Other platforms:** Send via the platform's message tool.

## CLI Reference

```bash
anygen <resource> <method> [flags]
```

| Flag | Description |
|------|-------------|
| `--params '<json>'` | URL/path parameters |
| `--data '<json>'` | Request body |
| `--dry-run` | Show the request without sending it |
| `--wait` | Re-poll until terminal state |
| `--timeout <ms>` | Polling timeout in milliseconds |

```bash
# Inspect a method's schema
anygen schema task.create
```
