# Brain Dump Automation

AI-powered personal knowledge management system that captures your scattered thoughts and automatically organizes them into a structured "Second Brain."

## Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Slack     │───▶│    n8n      │───▶│   Claude    │───▶│   Notion    │
│ #brain-dump │    │ (automation)│    │  (classify) │    │ (databases) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

**How it works:**
1. Dump unstructured thoughts into Slack (`#brain-dump`)
2. n8n catches the message and fetches thread context
3. Claude AI classifies and extracts structured data
4. Items are routed to the appropriate Notion database
5. Daily/weekly digests keep you informed

## Tech Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| Capture | Slack | Quick mobile-friendly input |
| Automation | n8n | Workflow orchestration |
| Intelligence | Claude 3.5 Sonnet | Classification & summarization |
| Storage | Notion | Structured databases |

## Workflows

### The Sorter (`n8n_workflow.json`)
Main workflow that captures, classifies, and routes your brain dumps.

### The Digest (`n8n_digest_workflow.json`)
- **Daily (Mon-Sat)**: Morning briefing of tasks and action items
- **Weekly (Sunday)**: Strategic review of people met and ideas logged

### The Fixer (`n8n_fixer_workflow.json`)
React with ❌ or 🗑️ to delete misclassified items.

## Quick Start

1. **Install n8n**
   ```bash
   docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n
   ```

2. **Set up Slack App** with `channels:history` permissions

3. **Create Notion databases**: Inbox, Tasks, Projects, Resources, People

4. **Import workflows** into n8n

5. **Configure API keys** for Claude and Notion

## Verification

Send two messages to `#brain-dump`:
1. "Meeting with Sarah about Q3."
2. "Remind me to send her the report."

If Notion shows a task "Send report to Sarah", your Second Brain is live.

## Documentation

- [Implementation Guide](implementation_guide.md) - Step-by-step setup
- [Prompts Specification](prompts_specification.md) - Claude system prompts
- [Design Review](design_review.md) - V2 architecture and improvements
- [Tech Stack Analysis](tech_stack_analysis.md) - Alternative stack options

## V2 Features

- **Thread Context Awareness**: Fetches conversation history so Claude understands pronouns like "that" and "her"
- **Confidence Scoring**: Low-confidence items go to Inbox for manual review
- **Emoji Corrections**: Delete/archive items with reactions

## License

MIT
