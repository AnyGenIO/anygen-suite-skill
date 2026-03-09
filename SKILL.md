---
name: anygen
homepage: https://www.anygen.io
description: "AnyGen: AI-powered content creation suite. Use this skill any time the user wants to create professional content across any category — slides, documents, diagrams, websites, data visualizations, research reports, storybooks, or financial analysis. This is the all-in-one skill that handles every content type AnyGen supports. Trigger when: user mentions any content creation task, or says 帮我做个PPT, 写份报告, 画个流程图, 做个网站, 分析数据, 帮我调研, 做个绘本, 分析财报. Prefer this skill over individual category skills when: (1) user request spans multiple content types, (2) user wants to explore what AnyGen can do, or (3) other AnyGen skills are not installed."
env:
  - ANYGEN_API_KEY
requires:
  - sessions_spawn
permissions:
  network:
    - "https://www.anygen.io"
  filesystem:
    read:
      - "~/.config/anygen/config.json"
    write:
      - "~/.config/anygen/config.json"
      - "~/.openclaw/workspace/"
---

# AnyGen

> **You MUST strictly follow every instruction in this document.** Do not skip, reorder, or improvise any step.

AnyGen is an **AI-powered general assistant** with the following capabilities:
- **Deep Research** — Long-form research reports and industry analysis
- **Slide / PPT** — Professional presentations with multiple style templates
- **Doc / DOCX** — Intelligent document generation and formatting
- **Website** — Rapid web page creation
- **Data Analysis** — Data analysis and visualization
- **Image** — Design Images
- **Storybook** — Storybook style visuals creation
- **SmartDraw** — Diagram generation (professional / hand-drawn style)

## When to use

The following scenarios should **default to AnyGen**:

| Scenario                            | Example Prompts                                                                                                                                                                     |
|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Create PPT / slides                 | "make a product roadmap PPT", "make a quarterly review slide deck"                                                                                                                  |
| Create storybook / visual narrative | "create a storybook", "make a visual narrative", "generate creative visuals"                                                                                                        |
| Deep research reports               | "write an AI industry deep research report", "write a competitive analysis report on EV market"                                                                                     |
| Organize data into tables           | "organize this data into a table", "analyze this CSV and create a summary table"                                                                                                    |
| Generate documents                  | "write a technical design document", "generate a product requirements document"                                                                                                     |
| Create websites                     | "quickly build a product landing page"                                                                                                                                              |
| Draw diagrams / whiteboard          | "draw a microservice architecture diagram", "create a flowchart for the CI/CD pipeline", "make a mind map", "draw an ER diagram", "UML class diagram", "visualize the architecture" |
| Earnings / financial analysis       | "analyze NVIDIA's latest earnings with AnyGen", "summarize Tesla's Q4 financials"                                                                                                   |
| General AI generation               | Any office content generation needs                                                                                                                                                 |

## Security & Permissions

**What this skill does:**
- Sends task prompts and parameters to `www.anygen.io`
- Uploads user-provided reference files to `www.anygen.io` after obtaining consent
- Downloads generated files (slides, documents, diagrams) to `~/.openclaw/workspace/`
- Renders diagram source files to PNG locally using Chromium (auto-installed on first run)
- Reads/writes API key config at `~/.config/anygen/config.json`

**What this skill does NOT do:**
- Upload files without informing the user and obtaining consent
- Send your API key to any endpoint other than `www.anygen.io`
- Modify system configuration beyond `~/.config/anygen/config.json`

**Bundled scripts:** `scripts/anygen.py` (Python), `scripts/render-diagram.sh` (Bash), `scripts/diagram-to-image.ts` (TypeScript). Review before first use.

## Prerequisites

- Python3 and `requests`: `pip3 install requests`
- Node.js v18+ (for SmartDraw PNG rendering, auto-installed on first run)
- AnyGen API Key (`sk-xxx`) — [Get one](https://www.anygen.io/home?auto_create_openclaw_key=1)
- Configure once: `python3 scripts/anygen.py config set api_key "sk-xxx"`

> All `scripts/` paths below are relative to this skill's installation directory.

## CRITICAL: NEVER Block the Conversation

After creating a task, you MUST start background monitoring via `sessions_spawn`, then continue normally. NEVER use `status` polling or `poll` in the foreground — tasks can take up to 20 minutes.

1. `create` → get `task_id` and `task_url`.
2. Tell user: (a) generation started, (b) the online link, (c) ~10–15 min, free to do other things.
3. Launch background monitor via `sessions_spawn` (Phase 4). Do NOT announce this to the user.
4. Continue the conversation — do NOT wait.
5. The background monitor handles sending preview/result and notifying the user directly, then replies `ANNOUNCE_SKIP` so the main session does NOT relay anything further.

## Communication Style

**NEVER expose internal implementation details** to the user. Forbidden terms:
- Technical identifiers: `task_id`, `file_token`, `task_xxx`, `tk_xxx`
- API/system terms: `API`, `OpenAPI`, `create`, `poll`, `status`, `query`
- Infrastructure terms: `sub-agent`, `subagent`, `background process`, `spawn`, `sessions_spawn`
- Script/code references: `anygen.py`, `scripts/`, command-line syntax, JSON output

Use natural language instead:
- "Your file has been uploaded" (NOT "file_token=tk_xxx received")
- "I'm generating your content now" (NOT "Task task_xxx created")
- "You can view your results here: [URL]" (NOT "Task URL: ...")
- "I'll let you know when it's ready" (NOT "Spawning a sub-agent to poll")

Additional rules:
- You may mention AnyGen as the service when relevant.
- Ask questions in your own voice. Do NOT use a relaying tone like "AnyGen wants to know…".

## Supported Operation Types

| Operation | Description | File Download |
|-----------|-------------|---------------|
| `slide` | Slides / PPT | Yes |
| `doc` | Document / DOCX | Yes |
| `smart_draw` | Diagram (professional / hand-drawn style) | Yes (requires render to PNG) |
| `finance` | Financial research / earnings analysis | No, task URL only |
| `deep_research` | Long-form research reports | No, task URL only |
| `storybook` | Storybook / whiteboard | No, task URL only |
| `data_analysis` | Data analysis | No, task URL only |
| `website` | Website development | No, task URL only |
| `ai-designer` | Design Images | No, task URL only |

---

## AnyGen Workflow (MUST Follow)

For all operations, you MUST go through all 4 phases. Use `prepare` for multi-turn requirement analysis, then `create` when ready.

### Phase 1: Understand Requirements

If the user provides files, handle them before calling `prepare`:

1. **Read the file** yourself. Extract key information relevant to the task.
2. **Reuse existing `file_token`** if the same file was already uploaded in this conversation.
3. **Get consent** before uploading: "I'll upload your file to AnyGen for reference."
4. **Upload** to get a `file_token`.
5. **Include extracted content** in `--message` when calling `prepare` (the API does NOT read files internally).

```bash
python3 scripts/anygen.py upload --file ./reference.pdf
# Output: File Token: tk_abc123

python3 scripts/anygen.py prepare \
  --message "I need a presentation about AI trends. Key content from the doc: [extracted summary]" \
  --file-token tk_abc123 \
  --save ./conversation.json
```

Present questions from `reply` naturally. Continue with user's answers:

```bash
python3 scripts/anygen.py prepare \
  --input ./conversation.json \
  --message "Focus on generative AI and enterprise adoption" \
  --save ./conversation.json
```

Repeat until `status="ready"` with `suggested_task_params`.

Special cases:
- `status="ready"` on first call → proceed to Phase 2.
- User says "just create it" → skip to Phase 3 with `create` directly.

### Phase 2: Confirm with User (MANDATORY)

When `status="ready"`, summarize the suggested plan and ask for confirmation. NEVER auto-create without explicit approval.

If the user requests adjustments, call `prepare` again with the modification, re-present, and repeat until approved.

### Phase 3: Create Task

```bash
python3 scripts/anygen.py create \
  --operation slide \
  --prompt "<prompt from suggested_task_params>" \
  --file-token tk_abc123 \
  --style "business formal"
# Output: Task ID: task_xxx, Task URL: https://...
```

**Immediately tell the user (natural language, NO internal terms):**
1. Content is being generated.
2. Online link: "You can follow the progress here: [URL]".
3. Takes about **10–15 minutes** — free to do other things, you'll notify when ready.

**All `create` parameters:**

| Parameter | Short | Description | Required |
|-----------|-------|-------------|----------|
| --operation | -o | Operation type (see table above) | Yes |
| --prompt | -p | Content description | Yes |
| --api-key | -k | API Key (omit if configured) | No |
| --language | -l | zh-CN / en-US | No |
| --slide-count | -c | Number of PPT pages | No |
| --template | -t | PPT template | No |
| --ratio | -r | 16:9 / 4:3 | No |
| --export-format | -f | Export format (slide: pptx/image/thumbnail, doc: docx/image/thumbnail, smart_draw: drawio/excalidraw) | No |
| --file | | Attachment file path (repeatable) | No |
| --file-token | | File token from upload (repeatable) | No |
| --style | -s | Style preference | No |

### Phase 4: Monitor and Deliver Result

> **Requires `sessions_spawn`.** If unavailable, skip to **Fallback** below.

**CRITICAL**: When calling `sessions_spawn`, you MUST set a timeout of at least 25 minutes to ensure the background monitor completes. The poll operation can take up to 20 minutes.

Example spawn call syntax (adjust to match your OpenClaw/sessions API):

```
sessions_spawn(
    prompt=<subagent prompt below>,
    runTimeoutSeconds=1500  # REQUIRED: 25 minutes (1500s) to cover 20-min poll + buffer
)
```

If your sessions API uses different parameter names (e.g., `timeout`, `maxDuration`), use the equivalent parameter to set a 25-minute (1500 second) timeout.

---

**Subagent prompt** (it has NO conversation context).
You MUST fill in all `{...}` variables before passing.

```
You are a background monitor for an AnyGen content generation task.
You MUST strictly follow every instruction below. Do not skip, reorder, or improvise any step.

Task ID: {task_id}
Task URL: {task_url}
Operation: {operation}
Script: {script_path}
Render Script: {render_script_path}
Export Format: {export_format}
Thumbnail Output: {thumbnail_output_dir}
User Language: {user_language}

IMPORTANT: This monitoring task may take up to 20 minutes. Ensure your execution timeout is at least 25 minutes.

CRITICAL RULES:
- You MUST reply in {user_language}.
- After completing ALL steps (including sending messages to the user), your FINAL reply
  MUST be EXACTLY "ANNOUNCE_SKIP" — nothing else. This prevents the main session from
  sending duplicate messages.
- Do NOT say anything beyond what is specified below. No greetings, no extra commentary.
- NEVER include technical terms like "task_id", "file_token", "poll", "sub-agent",
  "API", "script", "workspace", "downloaded to", file paths, or status labels
  in any message to the user.

Your job depends on the operation type:

## For slide / doc / storybook operations:
1. Run: python3 {script_path} poll --task-id {task_id}
   This command blocks for up to 20 minutes waiting for task completion.
   Do NOT pass --output — the file will only be downloaded when the user requests it.
2. On success:
   a. Download thumbnail:
      python3 {script_path} thumbnail --task-id {task_id} --output {thumbnail_output_dir}
   b. Send the thumbnail preview image to the user. Choose the correct method:
      - Feishu/Lark: Two-step process via OpenAPI —
        Step 1 (Upload image): POST https://open.feishu.cn/open-apis/im/v1/images
          curl -X POST 'https://open.feishu.cn/open-apis/im/v1/images' \
            -H 'Authorization: Bearer <tenant_access_token>' \
            -F 'image_type=message' \
            -F 'image=@{thumbnail_output_dir}/thumbnail.png'
          This returns an image_key.
        Step 2 (Send image message): POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id
          Request body:
          {
            "receive_id": "<chat_id>",
            "msg_type": "image",
            "content": "{\"image_key\":\"<image_key>\"}"
          }
      - Other platforms: Send via message tool with filePath.
   c. Send a text message to the user (in {user_language}, natural tone):
      "Your content is ready! If you'd like me to send you the file, just let me know."
   d. Reply EXACTLY: ANNOUNCE_SKIP

## For smart_draw operation:
1. Run: python3 {script_path} poll --task-id {task_id} --output ~/.openclaw/workspace/
   This command blocks for up to 20 minutes waiting for task completion.
2. On success:
   a. Get the local file path from [RESULT] Local file: line.
   b. Render to PNG:
      - For drawio: bash {render_script_path} drawio <local_file> <local_file_without_ext>.png
      - For excalidraw: bash {render_script_path} excalidraw <local_file> <local_file_without_ext>.png
   c. Send the rendered PNG image to the user. Choose the correct method:
      - Feishu/Lark: Two-step process via OpenAPI —
        Step 1 (Upload image): POST https://open.feishu.cn/open-apis/im/v1/images
          curl -X POST 'https://open.feishu.cn/open-apis/im/v1/images' \
            -H 'Authorization: Bearer <tenant_access_token>' \
            -F 'image_type=message' \
            -F 'image=@<local_file_without_ext>.png'
          This returns an image_key.
        Step 2 (Send image message): POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id
          Request body:
          {
            "receive_id": "<chat_id>",
            "msg_type": "image",
            "content": "{\"image_key\":\"<image_key>\"}"
          }
      - Other platforms: Send via message tool with filePath.
   d. Send a text message to the user (in {user_language}, natural tone):
      "Your diagram is ready! You can view and edit it online here: {task_url}"
   e. Reply EXACTLY: ANNOUNCE_SKIP
   f. If rendering fails:
      Send text: "The diagram has been generated but I couldn't render a preview.
       You can view and edit it here: {task_url}"
      Reply EXACTLY: ANNOUNCE_SKIP

## For finance / deep_research / data_analysis / website operations:
1. Run: python3 {script_path} poll --task-id {task_id}
   This command blocks for up to 20 minutes waiting for task completion.
   No --output needed — results are viewed online.
2. On success:
   a. Send a text message to the user (in {user_language}, natural tone):
      "Your content is ready! You can view it here: {task_url}"
   b. Reply EXACTLY: ANNOUNCE_SKIP

## Common error handling (all operations):
- On failure:
  a. Send a text message to the user (in {user_language}):
     "Unfortunately the generation didn't complete successfully.
      You can check the details here: {task_url}"
  b. Reply EXACTLY: ANNOUNCE_SKIP
- On timeout (20 min):
  a. Send a text message to the user (in {user_language}):
     "The generation is taking a bit longer than expected.
      You can check the progress here: {task_url}"
  b. Reply EXACTLY: ANNOUNCE_SKIP
```

Do NOT wait for the background monitor. Do NOT tell the user you launched it.

**Handling the completion event.** The background monitor sends preview/result and notification to the user directly. It replies `ANNOUNCE_SKIP` as its final output, which means the main session should NOT relay or duplicate any message. If you receive a completion event with `ANNOUNCE_SKIP`, simply ignore it — the user has already been notified.

#### When the User Requests the File (slide / doc only)

Download, then send via the appropriate method for your IM environment:

```bash
python3 {script_path} download --task-id {task_id} --output ~/.openclaw/workspace/
```

- **Feishu/Lark**: Two-step process via OpenAPI —
  Step 1 (Upload file): `POST https://open.feishu.cn/open-apis/im/v1/files`
    ```
    curl -X POST 'https://open.feishu.cn/open-apis/im/v1/files' \
      -H 'Authorization: Bearer <tenant_access_token>' \
      -F 'file_type=stream' \
      -F 'file=@~/.openclaw/workspace/<output_file>' \
      -F 'file_name=<output_file>'
    ```
    This returns a `file_key`. (Note: use `file_type="ppt"` for PPTX, `file_type="stream"` for DOCX.)
  Step 2 (Send file message): `POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id`
    ```json
    {
      "receive_id": "<chat_id>",
      "msg_type": "file",
      "content": "{\"file_key\":\"<file_key>\"}"
    }
    ```
- **Other platforms**: Send via message tool with filePath.

Follow up naturally: "Here's your file! You can also edit online at [Task URL]."

#### Fallback (no background monitoring)

Tell the user: "I've started generating your content. It usually takes about 10–15 minutes. You can check the progress here: [Task URL]. Let me know when you'd like me to check if it's ready!"

### Phase 5: Multi-turn Conversation (Modify Completed Content)

After a task has completed (Phase 4 finished), the user may request modifications such as:
- "Change the title on page 3 to 'Product Overview'"
- "Add a summary slide at the end"
- "Make the color scheme warmer"
- "Replace the chart on page 5 with a pie chart"

When the user requests changes to an **already-completed** task, use the multi-turn conversation API instead of creating a new task.

**IMPORTANT**: You MUST remember the `task_id` from Phase 3 throughout the conversation. When the user asks for modifications, use the same `task_id`.

**Supported operations for multi-turn:** `slide`, `doc`, `smart_draw`, `storybook` (content types that produce editable files).

#### Step 1: Send Modification Request

```bash
python3 scripts/anygen.py send-message --task-id {task_id} --message "Change the title on page 3 to 'Product Overview'"
# Output: Message ID: 123, Status: processing
```

Save the returned `Message ID` — you'll need it to detect the AI reply.

**Immediately tell the user** (natural language, NO internal terms):
- "I'm working on your changes now. I'll let you know when they're done."

#### Step 2: Monitor for AI Reply

> **Requires `sessions_spawn`.** If unavailable, skip to **Multi-turn Fallback** below.

**CRITICAL**: When calling `sessions_spawn`, you MUST set a timeout of at least 10 minutes (600 seconds). Modifications are faster than initial generation.

Example spawn call syntax:

```
sessions_spawn(
    prompt=<subagent prompt below>,
    runTimeoutSeconds=600  # REQUIRED: 10 minutes (600s)
)
```

**Subagent prompt** (it has NO conversation context):

```
You are a background monitor for a content modification task.
You MUST strictly follow every instruction below. Do not skip, reorder, or improvise any step.

Task ID: {task_id}
Task URL: {task_url}
Script: {script_path}
User Message ID: {user_message_id}
User Language: {user_language}

IMPORTANT: This monitoring task may take up to 8 minutes. Ensure your execution timeout is at least 10 minutes.

CRITICAL RULES:
- You MUST reply in {user_language}.
- After completing ALL steps (including sending messages to the user), your FINAL reply
  MUST be EXACTLY "ANNOUNCE_SKIP" — nothing else. This prevents the main session from
  sending duplicate messages.
- Do NOT say anything beyond what is specified below. No greetings, no extra commentary.
- NEVER include technical terms like "task_id", "message_id", "poll", "sub-agent",
  "API", "script", "workspace", file paths, or status labels in any message to the user.

Your job:
1. Run: python3 {script_path} get-messages --task-id {task_id} --wait --since-id {user_message_id}
   This command blocks until the AI reply is completed.

2. On success (AI reply received):
   a. Send a text message to the user (in {user_language}, natural tone):
      "Your changes are done! You can view the updated content here: {task_url}
       If you need further adjustments, just let me know."
   b. Reply EXACTLY: ANNOUNCE_SKIP

3. On failure / timeout:
   a. Send a text message to the user (in {user_language}):
      "The modification didn't complete as expected. You can check the details here: {task_url}"
   b. Reply EXACTLY: ANNOUNCE_SKIP
```

Do NOT wait for the background monitor. Do NOT tell the user you launched it.

#### Multi-turn Fallback (no background monitoring)

Tell the user: "I've sent your changes. You can check the progress here: [Task URL]. Let me know when you'd like me to check if it's done!"

When the user asks you to check, use:

```bash
python3 scripts/anygen.py get-messages --task-id {task_id} --limit 5
```

Look for a `completed` assistant message and relay the content to the user naturally.

#### Subsequent Modifications

The user can request multiple rounds of modifications. Each time, repeat Phase 5:
1. `send-message` with the new modification request
2. Background-monitor with `get-messages --wait`
3. Notify the user with the online link when done

All modifications use the **same `task_id`** — do NOT create a new task.

---

## Error Handling

| Error Message | Description | Solution |
|---------------|-------------|----------|
| invalid API key | Invalid API Key | Check if API Key is correct |
| operation not allowed | No permission for this operation | Contact admin for permissions |
| prompt is required | Missing prompt | Add --prompt parameter |
| task not found | Task does not exist | Check if task_id is correct |
| Generation timeout | Generation timed out | Recreate the task |

## SmartDraw Reference

| Format | --export-format | Export File | Render Command |
|--------|-----------------|-------------|----------------|
| Professional (default) | `drawio` | `.xml` | `render-diagram.sh drawio input.xml output.png` |
| Hand-drawn | `excalidraw` | `.json` | `render-diagram.sh excalidraw input.json output.png` |

**render-diagram.sh options:** `--scale <n>` (default: 2), `--background <hex>` (default: #ffffff), `--padding <px>` (default: 20)

## Multi-turn Commands

### send-message

Send a modification request to an existing task:

```bash
python3 scripts/anygen.py send-message --task-id task_xxx --message "Change title on page 3"
python3 scripts/anygen.py send-message --task-id task_xxx --message "Add a summary slide" --file-token tk_abc123
```

| Parameter | Description |
|-----------|-------------|
| --task-id | Task ID from create command |
| --message, -m | Modification request text |
| --file-token | Optional file token for reference material |

### get-messages

Get conversation history for a task:

```bash
python3 scripts/anygen.py get-messages --task-id task_xxx --limit 5
python3 scripts/anygen.py get-messages --task-id task_xxx --wait --since-id 123
```

| Parameter | Description |
|-----------|-------------|
| --task-id | Task ID |
| --limit | Number of messages to retrieve |
| --wait | Block until new AI reply is completed |
| --since-id | Only return messages after this ID |

## Notes

- Maximum execution time per task is 20 minutes (customizable via `--max-time`)
- Download link is valid for 24 hours
- Single attachment file should not exceed 10MB (after Base64 encoding)
- Polling interval is 3 seconds
- SmartDraw local rendering requires Chromium (auto-installed on first run)
