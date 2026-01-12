# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Brain Dump Automation is an AI-powered personal knowledge management system. It captures unstructured thoughts from Slack and automatically organizes them into Notion databases using Claude AI for classification.

**Architecture**: Slack → n8n → Claude AI → Notion

## Key Files

| File | Purpose |
|------|---------|
| `n8n_workflow.json` | Main brain dump processor (vision, bouncer, classification) |
| `n8n_digest_workflow.json` | Daily briefing and weekly strategic review |
| `n8n_fixer_workflow.json` | Quick corrections via Slack reactions |
| `prompts_specification.md` | 7 Claude system prompts (Sorter, Bouncer, Digest, etc.) |
| `implementation_guide.md` | Full V2.3 setup and configuration guide |

## Deployment

```bash
# Launch n8n with Docker
docker run -it --rm --name n8n -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e GENERIC_TIMEZONE="America/New_York" \
  n8nio/n8n

# Access at http://localhost:5678
# Import the three workflow JSON files
```

## n8n Environment Variables

Configure in n8n Settings → Variables:

**Required:**
- `NOTION_DB_INBOX`, `NOTION_DB_TASKS`, `NOTION_DB_RESOURCES`, `NOTION_DB_PEOPLE` (32-char database IDs)

**Optional:**
- `NOTION_DB_LOGS` - Audit trail database
- `CONFIDENCE_THRESHOLD` - Route to Inbox below this (default: 0.7)
- `THREAD_CONTEXT_LIMIT` - Max messages for context (default: 10)
- `WEEKLY_REVIEW_DAY` - 0=Sun, 1=Mon, etc. (default: 0)

## Processing Pipeline

```
Message → Validate → Duplicate Check → Haiku Bouncer → Thread Context → Sonnet Classifier → Route → Log
```

**Key behaviors:**
- Vision/OCR: Images/PDFs converted to base64 for Claude Vision
- Bouncer: Haiku pre-filter rejects noise ("lol", "thanks") at ~$0.00025/check
- Duplicate detection: MD5 hash checking against last 24 hours
- Thread capture: `/capture-thread` command combines entire thread into one item
- Confidence routing: Score < threshold → Inbox for manual review

## Verification Tests

1. **Basic**: Send "Meeting with Sarah about Q3" → Check Notion task created
2. **Context**: Msg1 "Meeting with Sam", Msg2 "Email him the docs" → Task should say "Email Sam"
3. **Vision**: Send image with caption → Notion includes image description
4. **Duplicate**: Same message twice → Second shows duplicate warning
5. **Bouncer**: Send "lol" → Filtered (not saved)
6. **Fixer**: React to receipt with ❌ → Notion page archived

## API Credentials Required

- **Slack**: Bot with `channels:history`, `chat:write`, `reactions:read`, `files:read`
- **Notion**: Integration token shared with each database
- **Anthropic**: API key in n8n header as `x-api-key`

## Notion Databases

- **DB_INBOX**: Low-confidence items for manual review
- **DB_TASKS**: Actionable items with deadlines, Status field
- **DB_RESOURCES**: Reference materials, ideas, content
- **DB_PEOPLE**: Contacts with company, email, last contact
- **DB_LOGS**: Audit trail (timestamp, hash, category, processing time)
