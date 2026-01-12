# Builder Stack Implementation Guide (V2.0 - Context Aware)

**Stack**: Slack (Capture) → n8n (Automation) → Claude 3.5 Sonnet (Intelligence) → Notion (Storage)

## 1. Stack Overview
*   **Capture**: Slack (Free Workspace). Fast, mobile-ready, reliable API.
*   **Automation**: n8n (Self-hosted or Cloud). Visual, infinite customization.
*   **Intelligence**: Claude 3.5 Sonnet (via API). Best-in-class reasoning.
*   **Storage**: Notion (Free/Personal Plan).

## 2. Setup Prerequisites
1.  **Slack**: Create a private channel `#brain-dump`. Create a Slack App and get the "Bot User OAuth Token". **Add `channels:history` and `groups:history` scopes.**
2.  **Notion**: Create an Integration and share your "Second Brain" page.
3.  **Anthropic**: Get an API Key.
4.  **n8n**: Set up an n8n instance.

## 3. Notion Database Structure
Create a parent page "Second Brain" containing these databases:
1.  **DB_Inbox**: For items with low confidence or manual review needed.
2.  **DB_Tasks**: Columns: `Status`, `Due Date`, `Project (Relation)`.
3.  **DB_Projects**: Columns: `Status`, `Deadline`, `Related Tasks`.
4.  **DB_Resources**: Columns: `Tags`, `URL`, `Summary`.
5.  **DB_People**: Columns: `Company`, `Last Contact`, `Phone/Email`.

## 4. n8n Workflow Design ("The Sorter")
*Updated to fetch Thread Context so the AI can understand references like "Add him to that project".*

### Node 1: Webhook (or Slack Trigger)
*   **Type**: `Slack Trigger`
*   **Settings**: Event = "New Message Posted to Channel", Channel = `#brain-dump`.
*   **Output**: Captures `Message ID` (`ts`) and `Thread TS` (`thread_ts`).

### Node 2: Slack (Fetch Context)
*   **Type**: `Slack`
*   **Operation**: `Message` -> `Get Many` (or `Get Replies`)
*   **Channel**: `#brain-dump`
*   **Timestamp/Thread**: Use the `thread_ts` from Node 1 (if it exists) or just the `ts`.
*   **Limit**: Last 3 messages.
*   **Goal**: Create a string variable `context_history` containing the last 3 user messages.

### Node 3: HTTP Request (Claude API)
*   **Method**: `POST`
*   **URL**: `https://api.anthropic.com/v1/messages`
*   **Headers**: `x-api-key: [YOUR_KEY]`, `anthropic-version: 2023-06-01`, `content-type: application/json`
*   **Body**:
    ```json
    {
      "model": "claude-3-5-sonnet-20240620",
      "max_tokens": 1000,
      "system": "[INSERT PROMPT FROM prompts_specification.md]",
      "messages": [
        {"role": "user", "content": "Context History:\n{{$json.context_history}}\n\nCurrent Message:\n{{$json.text}}"}
      ]
    }
    ```
*   **Output Parsing**: Use a `Code` node to `JSON.parse` the content.

### Node 4: Switch (Routing)
*   **Mode**: Expression (Same as V1)
    *   **Route 0 (Low Confidence)**: `confidence_score < 0.7` -> Go to **Inbox Node**.
    *   **Route 1 (Task)**: `category == "Project"` OR `subcategory == "Tasks"` -> Go to **Task Node**.
    *   **Route 2 (Resource)**: `category == "Resource"` -> Go to **Resource Node**.
    *   **Route 3 (People)**: `subcategory == "People"` -> Go to **People Node**.

### Node 5: Notion Nodes
*   **Same as V1**: Create pages in `DB_Inbox`, `DB_Tasks`, etc.

### Node 6: Slack (The Receipt)
*   **Action**: `Post Message`
*   **Channel**: `#brain-dump` (Thread reply)
*   **Text**: "✅ Filed as **{{$json.category}}** in Notion."

## 5. Verification
1.  **Test**:
    *   Msg 1: "Meeting with Sam from Engineering about the API."
    *   Msg 2: "Remind me to email **him** the docs."
2.  **Verify**:
    *   Check Notion `DB_Tasks`. Does the new task say "Email **Sam**" instead of "Email him"?
    *   If yes, Context works.

## 6. Daily Digest Workflow ("The Tap on the Shoulder")
*A separate n8n workflow to summarize your day.*

1.  **Import**: Use the updated `n8n_digest_workflow.json` file.
2.  **Logic**: It checks `{{ $today.weekday }}`.
    *   **Mon-Sat**: Runs "Daily Briefing" (Focus on Tasks).
    *   **Sunday**: Runs "Weekly Strategic Review" (Focus on People & Ideas).
3.  **Notion Nodes**: Ensure you map the `DB_PEOPLE_ID` and `DB_RESOURCES_ID` for the Sunday path.
4.  **Prompt**: Copy the **Weekly Review** prompt from `prompts_specification.md` into the second Claude node.
5.  **Output**: Posts the relevant summary to Slack.

## 7. The Fixer Workflow (Correction)
*Allows you to delete items by reacting with an emoji.*

1.  **Import**: `n8n_fixer_workflow.json`
2.  **Trigger**: Slack `event:reaction_added`.
3.  **Setup**:
    *   Ensure your Slack App has `reactions:read` scope.
    *   Reinstall the app to your workspace.
4.  **Usage**: React with ❌ or 🗑️ to a "Receipt" message from the bot.
5.  **Result**: The bot archives the Notion page and confirms deletion.
