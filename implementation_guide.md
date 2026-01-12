# Builder Stack Implementation Guide (V2.3 - Full Featured)

**Stack**: Slack (Capture) → n8n (Automation) → Claude 3.5 Sonnet (Intelligence) → Notion (Storage)

---

## Table of Contents
1. [Stack Overview](#1-stack-overview)
2. [Setup Prerequisites](#2-setup-prerequisites)
3. [Environment Variables](#3-environment-variables)
4. [Notion Database Structure](#4-notion-database-structure)
5. [Main Workflow Features](#5-main-workflow-features)
6. [Digest & Weekly Review](#6-digest--weekly-review)
7. [The Fixer Workflow](#7-the-fixer-workflow)
8. [Verification & Testing](#8-verification--testing)
9. [Backup & Recovery](#9-backup--recovery)

---

## 1. Stack Overview

| Component | Tool | Purpose |
|-----------|------|---------|
| **Capture** | Slack (Free Workspace) | Fast, mobile-ready, reliable API |
| **Automation** | n8n (Self-hosted or Cloud) | Visual workflow builder, infinite customization |
| **Intelligence** | Claude 3.5 Sonnet + Haiku | Best-in-class reasoning + cost-effective filtering |
| **Storage** | Notion (Free/Personal Plan) | Flexible database with relations |

### V2.3 Features
- **Vision/OCR Support**: Process images and PDFs attached to messages
- **Cheap Bouncer**: Pre-filter noise with Claude Haiku (~$0.00025/check)
- **Duplicate Detection**: MD5 hash-based deduplication within 24 hours
- **Thread Capture**: Capture entire threads as single items with `/capture-thread`
- **Configurable Settings**: Confidence threshold, thread context limit, weekly review day
- **Audit Logging**: Track all processed items in dedicated Notion database
- **Rate Limiting**: Prevent API throttling with built-in delays

---

## 2. Setup Prerequisites

### Slack App Configuration
1. Create a private channel `#brain-dump`
2. Create a Slack App at https://api.slack.com/apps
3. Add the following **Bot Token Scopes**:
   - `channels:history` - Read channel messages
   - `groups:history` - Read private channel messages
   - `channels:read` - View channel info
   - `chat:write` - Post messages
   - `reactions:read` - Read emoji reactions (for Fixer)
   - `files:read` - Access file attachments (for Vision)
4. Install the app to your workspace
5. Invite the bot to `#brain-dump`

### Notion Integration
1. Create an Integration at https://www.notion.so/my-integrations
2. Required Capabilities:
   - ✅ Read content
   - ✅ Update content
   - ✅ Insert content
3. Share each database with the integration (click "..." → "Connections" → Add your integration)
4. Copy each database ID from the URL (the 32-character string after the workspace name)

### Anthropic API
1. Get API Key from https://console.anthropic.com
2. Store securely - never commit to version control
3. Create n8n credential with header: `x-api-key: YOUR_KEY`

### n8n Setup
1. **Docker (Recommended)**:
   ```bash
   docker run -it --rm --name n8n -p 5678:5678 \
     -v ~/.n8n:/home/node/.n8n \
     -e GENERIC_TIMEZONE="America/New_York" \
     n8nio/n8n
   ```
2. **Cloud**: Sign up at https://n8n.io

---

## 3. Environment Variables

Configure these in n8n Settings → Variables:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NOTION_DB_INBOX` | Yes | - | Inbox database ID (32 chars) |
| `NOTION_DB_TASKS` | Yes | - | Tasks database ID |
| `NOTION_DB_RESOURCES` | Yes | - | Resources database ID |
| `NOTION_DB_PEOPLE` | Yes | - | People database ID |
| `NOTION_DB_LOGS` | No | - | Audit log database ID |
| `SLACK_ERROR_CHANNEL` | No | `brain-dump-errors` | Channel for error notifications |
| `SLACK_DIGEST_CHANNEL` | No | `brain-dump` | Channel for daily/weekly digests |
| `THREAD_CONTEXT_LIMIT` | No | `10` | Max messages to fetch for thread context |
| `CONFIDENCE_THRESHOLD` | No | `0.7` | Below this score → route to Inbox |
| `WEEKLY_REVIEW_DAY` | No | `0` | Day for weekly review (0=Sun, 1=Mon, etc.) |

---

## 4. Notion Database Structure

Create a parent page "Second Brain" containing these databases:

### DB_INBOX
For items with low confidence or requiring manual review.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Title | Title | Yes | Short summary of the item |
| Summary | Rich Text | Yes | 1-2 sentence description |
| Category | Select | Yes | Project, Area, Resource, Archive, Inbox |
| Subcategory | Select | No | People, Ideas, Tasks, Content |
| Confidence | Number | Yes | 0.0 - 1.0 AI confidence score |
| Original | Rich Text | No | Original Slack message text |
| Hash | Rich Text | No | MD5 hash for duplicate detection |
| Source | Rich Text | No | "slack" or "slack-vision" |
| Created | Created Time | Auto | Timestamp |

### DB_TASKS
For actionable items with deadlines.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Title | Title | Yes | Task summary |
| Summary | Rich Text | Yes | Task details |
| Status | Select | Yes | Options: To Do, In Progress, Done, Archived |
| Due Date | Date | No | Task deadline |
| Category | Select | No | Classification |
| Action Items | Rich Text | No | Extracted action items |
| People | Multi-select | No | Related people |
| Project | Relation | No | Link to DB_Projects |
| Confidence | Number | No | AI confidence score |
| Original | Rich Text | No | Original message text |
| Hash | Rich Text | No | Duplicate detection hash |

### DB_RESOURCES
For reference materials, ideas, and content.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Title | Title | Yes | Resource title |
| Summary | Rich Text | Yes | Description |
| Category | Select | No | Area of interest |
| Tags | Multi-select | No | Searchable tags |
| URL | URL | No | External link |
| Confidence | Number | No | AI confidence score |
| Original | Rich Text | No | Original message |
| Hash | Rich Text | No | Duplicate detection hash |

### DB_PEOPLE
For contacts and relationship tracking.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Name | Title | Yes | Person's full name |
| Notes | Rich Text | No | Context, interests, notes |
| Company | Rich Text | No | Organization |
| Last Contact | Date | No | Most recent interaction |
| Email | Email | No | Contact email |
| Phone | Phone | No | Contact phone |
| Projects | Relation | No | Related projects |
| Confidence | Number | No | AI confidence score |
| Context | Rich Text | No | How you met |

### DB_LOGS (Optional)
Audit trail for all processed messages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Timestamp | Title | Yes | ISO timestamp |
| Hash | Rich Text | Yes | Message MD5 hash |
| Category | Select | Yes | Final classification |
| Confidence | Number | Yes | AI confidence score |
| Page ID | Rich Text | Yes | Created Notion page ID |
| Processing Time | Number | No | Milliseconds to process |
| Source | Select | No | text, image, pdf, thread |

### Database Relationships
Set up these relations for cross-referencing:

1. **Tasks → Projects** (Relation): Link tasks to parent projects
2. **People → Projects** (Relation): Track who's involved in what
3. **Projects → Tasks** (Rollup): Count of related tasks

---

## 5. Main Workflow Features

### Workflow Overview
Import `n8n_workflow.json` - the main brain dump processor.

```
Slack Message
    │
    ▼
[Validate Input] ─── Empty? ──→ Skip
    │
    ▼
[Check /capture-thread] ─── Yes ──→ [Fetch Full Thread]
    │
    ▼
[Has Files?] ─── Yes ──→ [Vision Pipeline] ──→ [Merge Description]
    │
    ▼
[Check Duplicate] ─── Duplicate ──→ [Reply: Duplicate]
    │
    ▼
[Cheap Bouncer (Haiku)] ─── Noise ──→ [Reply: Filtered]
    │
    ▼
[Fetch Thread Context]
    │
    ▼
[Claude Sorter (Sonnet)] ─── Classify ──→ [Parse JSON]
    │
    ▼
[Route by Category + Confidence]
    │
    ├── Low Confidence ──→ DB_INBOX
    ├── Task/Project ──→ DB_TASKS
    ├── Resource ──→ DB_RESOURCES
    └── People ──→ DB_PEOPLE
    │
    ▼
[Send Receipt to Slack]
    │
    ▼
[Log to DB_LOGS]
```

### Vision/OCR Support
When images or PDFs are attached:
1. Downloads the file via Slack API
2. Converts to base64 for Claude Vision API
3. Generates description: "Describe this image/document in detail"
4. Merges description with message text for classification

Supported formats: PNG, JPG, JPEG, GIF, WEBP, PDF (first page)

### Cheap Bouncer (Haiku Pre-filter)
Before full Sonnet processing, Haiku evaluates if the message is worth saving:
- **Cost**: ~$0.00025 per check vs $0.003 for Sonnet
- **Filter Examples**: "lol", "thanks", "ok", "👍"
- **Pass Examples**: "Meeting with John about Q3", "Idea for new feature"

### Duplicate Detection
- Computes MD5 hash of message text
- Queries Inbox for matching hash in last 24 hours
- Skips processing and notifies user if duplicate found

### Thread Capture
Trigger: Message starts with `/capture-thread`
- Fetches ALL messages in the thread (not just 10)
- Combines into single context
- Creates one comprehensive Notion item

---

## 6. Digest & Weekly Review

Import `n8n_digest_workflow.json` for automated summaries.

### Schedule
- **Trigger**: Daily at 8:00 AM (configurable timezone)
- **Weekdays**: Daily Briefing (tasks, inbox items)
- **Weekly Review Day**: Strategic Review (people, ideas, patterns)

### Configurable Weekly Review Day
Set `WEEKLY_REVIEW_DAY` environment variable:
- `0` = Sunday (default)
- `1` = Monday
- `2` = Tuesday
- ... and so on

### Filtering Completed Items
The digest automatically excludes items with status:
- Done
- Archived
- Completed

### Empty Result Handling
If no items to report, posts a friendly message instead of calling Claude.

---

## 7. The Fixer Workflow

Import `n8n_fixer_workflow.json` for quick corrections.

### Usage
React to any bot "Receipt" message with:
- ❌ (`:x:`)
- 🗑️ (`:wastebasket:`)
- ✖️ (`:heavy_multiplication_x:`)
- ❎ (`:negative_squared_cross_mark:`)

### Result
1. Bot finds the Notion page ID from the receipt message
2. Archives the page in Notion
3. Confirms with a threaded reply

### URL Parsing
Handles multiple Notion URL formats:
- `notion.so/Page-Title-abc123def456`
- `notion.so/abc123def456`
- `notion.so/workspace/Page-abc123def456`

---

## 8. Verification & Testing

### Basic Flow Test
1. Send to `#brain-dump`: "Meeting with Sarah about Q3 budget"
2. Verify:
   - ✅ Receipt appears in thread
   - ✅ Item created in Notion (DB_TASKS or DB_PEOPLE)
   - ✅ Log entry in DB_LOGS (if configured)

### Context Awareness Test
1. Msg 1: "Meeting with Sam from Engineering"
2. Msg 2: "Remind me to email **him** the docs"
3. Verify: Task says "Email Sam" not "Email him"

### Vision Test
1. Send an image with caption: "Notes from whiteboard"
2. Verify: Notion entry includes image description

### Duplicate Detection Test
1. Send: "Test message for duplicate"
2. Send same message again within 1 minute
3. Verify: Second message shows "Possible duplicate detected"

### Cheap Bouncer Test
1. Send: "lol"
2. Verify: Message filtered (not saved to Notion)
3. Send: "Meeting tomorrow at 3pm"
4. Verify: Message processed normally

### Thread Capture Test
1. Create a thread with 5+ messages
2. Reply with: `/capture-thread`
3. Verify: Single Notion entry with all messages

### Fixer Test
1. React to a receipt with ❌
2. Verify: Notion page archived, confirmation posted

### Configuration Test
1. Set `CONFIDENCE_THRESHOLD=0.9`
2. Send ambiguous message
3. Verify: Routes to Inbox (lower confidence)

---

## 9. Backup & Recovery

### Manual Export
1. Notion: Settings → Export → Markdown & CSV
2. Frequency: Monthly recommended
3. Store in: Cloud storage or Git repository

### Disaster Recovery
1. Re-import Notion export
2. Re-import n8n workflow JSON files
3. Reconfigure credentials and environment variables

### Workflow Versioning
Keep copies of workflow JSON files in version control:
- `n8n_workflow.json`
- `n8n_digest_workflow.json`
- `n8n_fixer_workflow.json`

---

## Troubleshooting

### Common Issues

**"No Notion page ID found"**
- Check that the receipt message contains a valid Notion URL
- Verify Notion API credentials are correct

**"Claude API rate limit"**
- The workflow includes built-in rate limiting
- Check Anthropic console for usage limits

**"Empty confidence score"**
- Ensure Claude response is valid JSON
- Check for parsing errors in the Code node

**"Duplicate detected" for unique messages**
- Hash collision (rare) or similar message in last 24h
- Check DB_INBOX for the original

### Error Notifications
All workflow errors are sent to `SLACK_ERROR_CHANNEL` with:
- Error message
- Failed node name
- Original message (if available)

---

## Changelog

### V2.3 (Current)
- Vision/OCR support for images and PDFs
- Cheap Bouncer (Haiku) pre-filter
- Duplicate detection with MD5 hashing
- Thread capture (`/capture-thread`)
- Configurable confidence threshold
- Configurable thread context limit
- Configurable weekly review day
- Filter completed items from digest
- Notion URL validation
- Enhanced people extraction
- Audit logging to DB_LOGS

### V2.2
- Error handling with Slack notifications
- Input validation
- Notion field mappings
- Full system prompts from specification
- Thread context formatting
- Confidence score routing
- Empty digest handling
- Timezone support (America/New_York)
- Fixer regex improvements

### V2.0
- Context-aware processing
- Thread history fetching
- Daily and weekly digest separation

### V1.0
- Basic classification and routing
- Single message processing
