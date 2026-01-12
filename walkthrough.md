# "Second Brain" Builder Stack Walkthrough (V2.0)

You have selected the **Builder Stack** with **Thread Context Awareness**:
*   **Capture**: Slack (with History Access)
*   **Storage**: Notion
*   **Automation**: n8n
*   **Intelligence**: Claude 3.5 Sonnet

## 📂 Final Documentation

### 1. [Builder Stack Implementation Guide (V2)](./implementation_guide.md)
*   **New in V2**: The "Context Aware" workflow.
*   **Key Node**: `Slack > Get Replies` (Fetches the conversation surrounding your brain dump).
*   **Why**: Enables you to say "Add **that** to the project" and have the AI understand what "**that**" refers to.

### 2. [System Prompts Specification](./prompts_specification.md)
*   **Standard**: Use the specified JSON schemas for Claude.

### 3. [Design Review (V2 Features)](./design_review.md)
*   **Read this for**: Understanding why we added the context features (The "Pronoun Problem").

## 🚀 Launch Checklist
1.  **Install n8n**: `docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n`
2.  **Scopes**: Ensure your Slack App has `channels:history` permissions.
3.  **Build**: Follow the V2 Guide to draw your n8n workflow.
4.  **Digest**: Import `n8n_digest_workflow.json` to enable morning summaries.
5.  **Fixer**: Import `n8n_fixer_workflow.json` to enable ❌ reaction deleting.

## ✅ Verification
Send two messages in a row to `#brain-dump`:
1.  "Meeting with Sarah about Q3."
2.  "Remind me to send **her** the report."

If Notion shows a task "Send report to **Sarah**", your Second Brain is live.
