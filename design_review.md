# Design Review & Optimization

**Status**: ⚠️ Optimization Opportunities Found

This document outlines potential gaps in the current "Builder Stack" design and suggests specific improvements to make the system more robust.

## 1. Identified Gaps

### A. Context Blindness ("Statelessness")
*   **Issue**: The current prompt only sees the *current* message.
*   **Scenario**:
    *   *User*: "We need to fix the login bug." -> Filed as Task.
    *   *User*: "Assign it to John." -> ❌ Fails (Bot doesn't know what "it" refers to).
*   **Risk**: High friction; user must always speak in full, self-contained sentences.

### B. Media Blindness
*   **Issue**: The current prompt assumes text-only input.
*   **Scenario**: User shares a screenshot of a chart or a PDF invoice to `#brain-dump`.
*   **Result**: The automation passes an empty text string or a URL that Claude can't read, leading to "Inbox" junk data or errors.

### C. Error Silence
*   **Issue**: If Notion API is down or the n8n webhook times out.
*   **Result**: The user sends a thought, thinks it's saved, but it vanishes. No "Receipt" is sent, but no "Error" message is sent either.

### D. Cost Inefficiency
*   **Issue**: Sending *every* Slack message to Claude 3.5 Sonnet (approx $3/M tokens).
*   **Risk**: If you chat in that channel or say "Thanks", you pay for a premium inference.

## 2. Proposed "V2" Improvements

### ✅ Optimization 1: Thread Context (Fixes Gap A)
*   **Change**: In n8n, fetch the last 3 messages from the Slack thread *before* sending to Claude.
*   **Logic**: `messages = [previous_msg_2, previous_msg_1, current_msg]`
*   **Benefit**: "Assign it to John" now works because Claude sees the previous "Login bug" context.

### ✅ Optimization 2: Vision/OCR Support (Fixes Gap B)
*   **Change**: If `message.files` exists in the Slack payload:
    1.  n8n downloads the file.
    2.  n8n sends the image to Claude 3.5 Sonnet (which has Vision capabilities).
*   **Benefit**: "Save this receipt" + [Image] -> Auto-extracts amount and date.

### ✅ Optimization 3: The "Cheap Bouncer" (Fixes Gap D)
*   **Change**: Use a cheap model (e.g., GPT-4o-mini or Claude Haiku) as the first step.
*   **Prompt**: "Is this a task/memory worth saving? YES/NO."
*   **Benefit**: Filters out "Thanks!", "Test", or "Lol" for pennies, saving the expensive Sonnet calls for real work.

### ✅ Optimization 4: Error Catch Node (Fixes Gap C)
*   **Change**: Add an n8n `Error Trigger` node.
*   **Action**: If *any* node fails, post a message to `#brain-dump`: "⚠️ System Error: [Error Message]. Text was: [Original Text]".
*   **Benefit**: Zero data loss guarantee.
