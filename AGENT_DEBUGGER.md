# Agent Debugger

The **Agent Debugger** is a diagnostic tool that lets users load any recorded conversation and inspect every decision the agent made — step by step, with timing, token usage, knowledge sources, arguments, and observations — without leaving the Power Platform.

---

## Table of Contents

1. [Why This Feature Exists](#why-this-feature-exists)
2. [Overview](#overview)
3. [Prerequisites](#prerequisites)
4. [Required Permissions](#required-permissions)
5. [Getting Started — Filters](#getting-started--filters)
6. [Analysis View](#analysis-view)
   - [General Information](#general-information)
   - [Conversation Preview](#conversation-preview)
   - [Debug Information](#debug-information)
   - [Transcript JSON](#transcript-json)
7. [Connected and Child Agents](#connected-and-child-agents)
8. [How Data Is Fetched and Processed](#how-data-is-fetched-and-processed)
9. [Troubleshooting](#troubleshooting)

---

## Why This Feature Exists

### The debugging gap in modern Copilot Studio

Copilot Studio has evolved from a rule-based topic editor into an **AI-orchestrated agent platform**. Agents no longer follow a fixed decision tree — the large language model decides at runtime which topics to trigger, which actions to invoke, which knowledge sources to query, and whether to delegate work to a connected sub-agent. This shift brings tremendous flexibility, but it also introduces a class of problems that traditional chatbot debugging tools were never designed to handle.

**The built-in Test Canvas in Copilot Studio** lets you run a live conversation and see which topic was triggered. That is useful during authoring, but it falls short in several critical ways once agents are in production:

| Limitation | Impact |
|---|---|
| Live only — no access to past conversations | Cannot investigate what happened in a specific user conversation that has already ended |
| Topic-level visibility only | Does not show individual action arguments, observations, LLM thoughts, or step-level timing |
| No token visibility | Cannot see how many tokens a step consumed, or which step is driving up model costs |
| No knowledge-source traceability | Cannot see what the knowledge base returned vs what the model actually cited in its answer |
| No multi-agent drill-down | When a connected agent is invoked, there is no way to follow into its transcript from the parent |
| No performance breakdown | Cannot identify which step is causing slow responses without manually parsing JSON |

Meanwhile, Copilot Studio writes an extremely detailed trace log — over **30 distinct event types** covering every plan step, LLM reasoning thought, knowledge search, token count, MCP tool call, and connected-agent session — into the `conversationtranscripts` Dataverse table. This data exists, but it arrives as raw, chunked JSON across potentially many records and is practically impossible to interpret manually.

### What organisations actually need

When agents are used in business-critical processes — HR intake, customer service, IT helpdesk, sales qualification — a failed or unexpected conversation is not just a product curiosity. It is an incident that requires root-cause analysis. Administrators need to answer questions like:

- **"Why did the agent give the wrong answer?"** — Was the right knowledge source even searched? Was it returned but not cited? Did a connected agent fail silently?
- **"Why is the agent slow?"** — Which step is taking 15 seconds? Is it the knowledge retrieval, a slow Power Automate flow, or a model reasoning step?
- **"Did the agent do what I configured it to do?"** — Did it invoke the correct action with the correct parameters? What did the action return? Did it honour the topic's decision branches?
- **"What did the AI actually think?"** — Before invoking a tool, the orchestrator records its reasoning. Surfacing this reasoning is essential for safety reviews, alignment checks, and understanding non-deterministic behaviour.
- **"Why did the conversation end in a SystemError?"** — Which step failed, and with what error payload?
- **"What happened in the sub-agent?"** — The parent agent delegated to a connected Interview Agent. The user got an unexpected response. What did the Interview Agent actually do?

### Why this is especially important for multi-agent systems

Multi-agent architecture — where a parent orchestrator invokes specialised connected agents — is now a first-class pattern in Copilot Studio. Each connected agent runs its own conversation in its own transcript record, linked to the parent by a derived conversation ID (`parentConversationId_sessionId`). Without tooling, debugging a multi-agent conversation means:

1. Manually finding the parent transcript record in Dataverse
2. Parsing the raw JSON to find the `ConnectedAgentInitializeTraceData` event
3. Constructing the child conversation ID by hand
4. Querying for the child transcript separately
5. Repeating for each connected agent
6. Correlating step timings, observations, and errors across all of them

The Agent Debugger automates this entirely. It detects connected agents from the parent transcript, constructs child conversation IDs, fetches and parses child transcripts on demand, and presents the full multi-agent conversation as a unified, navigable view.

### The result

The Agent Debugger closes the gap between what Copilot Studio records and what administrators can actually see. It turns a raw, fragmented, multi-record JSON payload into an actionable diagnostic interface — enabling faster incident resolution, informed prompt and knowledge-base tuning, cost visibility through token tracking, and confidence that AI-orchestrated agents are behaving as intended in production.

### Who uses Agent Debugger and how

#### End users — reporting an issue

When a user has a bad conversation — wrong answer, missing information, unexpected error — they can share the **Conversation ID** from their session with a support team or IT helpdesk. This single ID is enough for an administrator to pull up the exact conversation in Agent Debugger and see precisely what the agent did during that exchange. No reproduction is needed. No guesswork. The transcript is already in Dataverse.

> **Workflow:** User reports issue → shares Conversation ID → administrator opens Agent Debugger → selects agent and pastes ID → clicks Analyze → full conversation and execution path are immediately available.

#### Support teams and CSK Administrators — incident investigation

Administrators use Agent Debugger as a **post-mortem tool** for specific reported conversations. Common investigation scenarios:

- **Wrong answer:** Check which knowledge sources were searched and what was returned. Verify whether the correct source was cited — or whether the model ignored a relevant result.
- **Unexpected topic trigger:** See the intent recognition result and which topic the orchestrator chose, and why.
- **Silent failure:** Find the red (failed) step in the Debug Information panel, inspect its observation for the error payload, and identify whether a flow, connector, or connected agent was the root cause.
- **Conversation ended abruptly:** Check the session outcome (`SystemError`, `Abandoned`) in General Info, then trace back to the step that caused it.

The **View JSON** link in the Conversation Preview header opens the raw Transcript JSON that can be copied and attached to support tickets or shared with Microsoft for escalation.

#### Makers — optimising agent quality and performance

Makers use Agent Debugger to iteratively improve their agents based on real conversation data rather than test-canvas assumptions:

- **Bottleneck identification:** The Debug Information panel shows execution time per step. A step taking 15+ seconds immediately stands out. Makers can pinpoint whether it is a slow Power Automate flow, a large knowledge search, a connected agent round-trip, or a multi-step reasoning chain.
- **Knowledge source quality:** For each bot response, the Debug Information panel shows which knowledge sources were searched, which were returned, and which were actually cited. If the agent is consistently ignoring a relevant document, it is a signal to adjust chunking strategy, metadata, or prompt configuration.
- **Step arguments and observations:** Makers can verify that actions are receiving the correct inputs (e.g., correct variable values being passed to a flow) and returning the expected outputs — without writing test scripts.
- **LLM reasoning review:** The "Thought" field on each step shows the orchestrator's reasoning before it selected that step. Makers can see whether the model is interpreting user intent correctly or making incorrect assumptions.
- **Connected agent quality:** When a connected specialist agent is invoked, its child transcript can be loaded directly from the card. Makers can see every step the child agent took and verify it completed its task correctly.
- **Token cost awareness:** Per-step token counts help makers identify which steps are expensive. A knowledge retrieval step returning massive chunks, or a custom prompt with a very long system message, will show clearly in the token breakdown.

#### IT administrators and CoE teams — governance and oversight

- **Audit and compliance:** Any conversation can be reviewed to confirm the agent behaved appropriately, did not disclose sensitive information, and followed configured guardrails.
- **Outcome analysis:** Session outcomes (`Resolved`, `Escalated`, `Abandoned`, `SystemError`) are visible immediately without querying Dataverse directly. Teams can use this to prioritise which conversations need follow-up.
- **Multi-environment support:** Because Agent Debugger reads from the Agent Inventory — which can cover agents across multiple environments — administrators can debug agents in development, test, and production environments from a single interface.

---

## Overview

When a conversation takes place in Copilot Studio, the platform records a detailed activity log — every message, every plan step triggered, every knowledge source searched, every action invoked — as a **Conversation Transcript** in Dataverse. The Agent Debugger reads that transcript and surfaces it in a structured, human-readable interface.

**What it shows:**

| Area | What you learn |
|---|---|
| Conversation Preview | Full chat exchange, rendered with markdown and adaptive cards |
| Debug Information | Step-by-step execution for the selected user message: every topic, action, knowledge search, and tool invoked — with thought, arguments, observation, token counts, and knowledge sources |
| Transcript JSON | Full raw transcript activities as syntax-highlighted, searchable JSON — opened via the **View JSON** link in the Conversation Preview header |
| General Information | Session count, turn count, outcome, duration, start time, channel, and AI model used |

**Key capabilities:**

- Debug **multi-agent conversations**: automatically detects connected and child agents, and lets you drill into their transcripts with a single click
- Inspect **step arguments and observations**: see exactly what inputs were passed to every action, tool, or knowledge source — and what it returned
- Review **token consumption**: prompt and completion token counts per step and per turn
- Correlate **knowledge sources**: see what was searched, what was returned, and what was actually cited in the answer
- Copy conversation IDs, step IDs, or any JSON for use in support tickets or bug reports

---

## Prerequisites

### 1. Agent Inventory must be synchronised

The Agent Debugger lists agents from the **Agent Inventory** in Dataverse. If an agent does not appear in the dropdown, its inventory record is missing or its transcript flag is not set.

See the Agent Inventory documentation for setup instructions:
**→ [AGENT\_INVENTORY.md](https://github.com/microsoft/Power-CAT-Copilot-Studio-Kit/blob/main/AGENT_INVENTORY.md)**

Specifically:
- The agent must have been synced to the `cat_agentdetails` Dataverse table.
- The `cat_istranscriptavailablecode` field on the agent record must be set to `1` (transcript available). This is set automatically during inventory sync when the agent has transcripts.

### 2. Conversation Transcripts must exist

Transcripts are written to the `conversationtranscripts` Dataverse table by the Power Virtual Agents runtime automatically. No additional configuration is required — as long as conversation logging is enabled in the Copilot Studio agent settings, transcripts will be available.

> **Note:** Transcripts may take a few minutes to appear after a conversation ends.

### 3. Security role

The user must have one of the following security roles in the target Dataverse environment:

| Role | Access |
|---|---|
| **CSK - Administrator** | Full access to Agent Debugger and all Copilot Studio Kit admin features |
| **System Administrator** | Full environment access (includes all Copilot Studio Kit tables) |

Users with only the **CSK - Maker** role cannot access Agent Debugger — they will not see it in the navigation.

For background on Copilot Studio Kit security roles, see the Agent Insights Hub documentation:
**→ [AGENT\_INSIGHTS\_HUB.md](https://github.com/microsoft/Power-CAT-Copilot-Studio-Kit/blob/main/AGENT_INSIGHTS_HUB.md)**

---

## Required Permissions

Agent Debugger reads data from three Dataverse tables. The **Dataverse connection** configured in the app must have **read access** to all of them. This applies even when debugging agents that belong to a different environment — the connection must be authenticated with an identity that has the appropriate permissions in that environment.

| Table | Logical Name | What it is used for |
|---|---|---|
| **Agent Details** | `cat_agentdetails` | List of agents and their environment metadata |
| **Conversation Transcripts** | `conversationtranscripts` | Raw conversation activity logs |
| **Bot Components** | `botcomponents` | Schema-name-to-display-name lookup for topics, actions, and tools |

### Minimum required privileges

The identity used by the Dataverse connector needs at least:

- **Read** on `cat_agentdetails`
- **Read** on `conversationtranscripts`
- **Read** on `botcomponents`

The **CSK - Administrator** security role grants all of these. Alternatively, a custom role with the above read privileges on those three tables is sufficient.

> **Cross-environment access:** If the agent being debugged lives in a *different* environment from the one the app is installed in, the Dataverse connection must be authenticated in that remote environment and have the same read permissions there.

---

## Getting Started — Filters

When you open Agent Debugger, you see a three-step cascading filter bar before any data is loaded.

```
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
│  Environment     │ → │  Agent           │ → │  Conversation ID     │
│  (dropdown)      │   │  (dropdown)      │   │  (search + dropdown) │
└──────────────────┘   └──────────────────┘   └──────────────────────┘
                                                           │
                                                   [ Analyze ]
```

### Filter 1 — Environment

Populated from the distinct environment names found in Agent Inventory. Selecting an environment narrows the Agent dropdown to only agents registered in that environment.

### Filter 2 — Agent

Shows all agents in the selected environment that have at least one conversation transcript (`cat_istranscriptavailablecode = 1`). Selecting an agent loads the most recent 50 conversations for the Conversation ID dropdown.

### Filter 3 — Conversation ID

- **Default:** Shows the 50 most recent unique conversations for the selected agent.
- **Typing in the box:** Triggers a full search across all transcripts for that agent (up to 100,000 records), allowing you to find older or specific conversations.

Once a Conversation ID is selected, the **Analyze** button becomes active. Click it to open the full analysis view.

---

## Analysis View

The analysis view is a two-column layout: the **Conversation Preview** (left, sticky) and the **Debug Information panel** (right). Above both columns is a General Information summary. The Conversation Preview header includes a **View JSON** link to open the full raw Transcript JSON.

```
┌──────────────────────────────────────────────────────────────┐
│  General Information                                          │
│  Sessions: 1   Turns: 7   Outcome: Escalated   Duration: 3m  │
├─────────────────────────┬────────────────────────────────────┤
│  Conversation Preview   │  Debug Information (right)         │
│  (left, sticky)         │                                    │
│                         │  Turn 1 — "Book a flight"          │
│  User: Book a flight    │   ├─ Booking Topic  0.4s           │
│  Bot:  Where would...   │   ├─ Lookup Flights 2.1s           │
│  ...                    │   └─ Adaptive Card  0.1s           │
│                         │                                    │
│  [View JSON]            │  Turn 2 — "London"                 │
│                         │   └─ ...                           │
└─────────────────────────┴────────────────────────────────────┘
```

### General Information

Displayed as summary cards at the top of the analysis view.

| Field | Description |
|---|---|
| **Sessions** | Number of conversation sessions (multiple sessions occur when a user returns to the same conversation after inactivity) |
| **Turns** | Number of user messages in the conversation |
| **Outcome** | Session outcome reported by the platform (e.g., Resolved, Escalated, Abandoned, SystemError) |
| **Duration** | Total conversation duration from first to last activity |
| **Start Time** | When the conversation began (local time) |
| **Channel** | Communication channel used (e.g., webchat, msteams) — shown when available |
| **Model** | AI model name used by the agent — shown when available |

The outcome badge uses the platform's `SessionInfo` trace at the end of the transcript. A `SystemError` outcome usually means a flow, connector, or orchestration error occurred — check the Debug Information panel for red (failed) steps.

---

### Debug Information

The Debug Information panel (right side) shows every step the agent's orchestrator executed for the selected user message, grouped by conversation turn. Click any user message in the Conversation Preview to load its steps here.

**Each step card shows:**
- **Icon + Name** — derived from the step's schema name or component metadata (e.g., `Booking Topic`, `Search Knowledge`, `ResumeUpload`)
- **Type badge** — Topic, Knowledge, Code, Connector Action, Tool, Flow, Custom Prompt, Deep Reasoning, Connected Agent, or Child Agent
- **Duration** — how long the step took (from `DynamicPlanStepFinished.executionTime`)
- **Outcome badge** — success (completed) or failed

**Clicking a step expands it to show:**
- **Thought** — the LLM's reasoning before invoking this step (e.g., *"The user wants to process two resumes. I'll send both to the Application-Intake-Agent simultaneously."*)
- **Arguments** — the inputs passed to the step (JSON)
- **Observation** — the result returned by the step (JSON)
- **Token usage** — prompt + completion token counts and model name (when available)
- **Knowledge sources** — what was searched, what was returned, what was cited

**Clicking a user message in the Conversation Preview** loads that turn's steps into the Debug Information panel, making it easy to correlate what the user said with what the agent did.

**Step type icons and colours:**

| Type | Colour | Examples |
|---|---|---|
| Topic | Blue | CustomTopic steps |
| Knowledge | Teal | KnowledgeSource steps |
| Code | Green | P:CodeTool steps |
| Connected Agent | Purple | InvokeConnectedAgentTaskAction |
| Child Agent | Purple | type="Agent" orchestration steps |
| Connector Action | Grey | Office365Outlook, Teams, SharePoint |
| Flow | Purple | InvokeFlowTaskAction |
| Tool / MCP | Purple | MCP servers, P:UniversalSearchTool |
| Custom Prompt | Blue | Prompt steps with prediction output |
| Deep Reasoning | Blue | P:ReasonerTool |
| Failed | Red | Any step with error outcome |

---

### Conversation Preview

The Conversation Preview panel shows the full conversation exchange as it appeared to the user.

**What is rendered:**
- **Bot and user messages** — formatted with full markdown support: headings, lists, tables, bold/italic, code blocks, blockquotes, links
- **Adaptive Cards** — rendered in read-only mode exactly as the user would have seen them (TextBlocks, ColumnSets, Images, Input fields, Action buttons)
- **OAuth / consent cards** — shown with the connection name and Allow/Cancel buttons (display only)
- **Suggested actions** — displayed as chips below the bot message
- **Feedback** — thumbs up/down and comment captured by the platform, shown below the bot's message
- **Knowledge references** — expandable list of sources the bot cited

**Interaction:**
- **Clicking a user message** selects it and loads that turn's execution steps into the Debug Information panel on the right, so you can immediately see what the agent did in response to that message.
- The **View JSON** link in the Conversation Preview card header opens the full Transcript JSON dialog.

> **Note:** Bot messages in Copilot Studio use markdown formatting. All markdown in the Conversation Preview is rendered — tables, headings, bullet lists, and inline styles are displayed as formatted content, not raw syntax.

---

### Transcript JSON

The **View JSON** link in the top-right corner of the Conversation Preview header opens a **Transcript JSON** dialog showing the full raw transcript activities.

Use this when:
- You need to inspect an event type not surfaced in the Debug Information panel
- You want to copy specific fields for a support ticket
- You are investigating unexpected behaviour in the parsed views

**Features:**
- Full syntax highlighting
- Search with keyword highlighting and next/prev navigation
- Copy-to-clipboard button for the full JSON

---

## Connected and Child Agents

Copilot Studio supports **multi-agent architectures** where a parent agent delegates tasks to other agents:

- **Connected agents** — separately published agents invoked via `InvokeConnectedAgentTaskAction`
- **Child agents** — agents called through the orchestration layer (`type = "Agent"` steps)

The Agent Debugger detects both kinds automatically from the parent transcript and surfaces them as special step cards in the Debug Information panel.

**Connected Agent card shows:**
- Agent name
- Total execution time
- The instruction/task passed to the child agent
- The child conversation ID (copyable)

**Loading the child transcript:**

Click the **Load** button on a Connected Agent card to fetch and parse the child agent's own transcript from Dataverse. Once loaded, the card expands to show:

- Each step the child agent executed (topics, actions, tools)
- Step-level execution times, arguments, and observations
- The child's own knowledge searches and responses

> This works because each connected agent session writes its transcript to Dataverse under a conversation ID derived from the parent conversation: `{parentConversationId}_{sessionId}`. The Agent Debugger constructs this ID automatically and queries for it.

---

## How Data Is Fetched and Processed

All data originates from a single source: **Dataverse conversation transcripts**. No live agent connection is made.

```mermaid
flowchart TD
    A([User opens Agent Debugger]) --> B[Load Agent List\ncat_agentdetails\nFilter: transcript available]
    B --> C[User selects Environment → Agent]
    C --> D[Load recent transcripts\nconversationtranscripts\nFilter by bot ID · Limit 50]
    D --> E[User selects Conversation ID\nand clicks Analyze]

    E --> F[Fetch all transcript records\nfor that conversation ID\nconversationtranscripts\nFilter: name contains conversationId\nAND bot_conversationtranscriptid = botId]

    F --> G[Merge multi-record chunks\nGroup by session start time\nSort by BatchId metadata\nConcatenate activities]

    G --> H[Parse transcript\n30+ activity/event types\nBuild ProcessedTurn array]

    H --> I[Lookup bot component names\nbotcomponents table\nSchema names → display names]

    H --> J[Detect connected agents\nConnectedAgentInitializeTraceData\nDynamicPlanStepTriggered\nDynamicPlanStepFinished]

    I --> K[Render Analysis View]
    J --> K

    K --> L{User clicks\nLoad on\nconnected agent?}

    L -->|Yes| M[Fetch child transcript\nconversationtranscripts\nFilter: name contains childConversationId]
    M --> N[Parse child transcript\nBuild child topics from steps]
    N --> O[Populate Connected Agent Card\nwith child steps and observations]
    O --> K

    L -->|No| P([User continues exploring])
    K --> P
```

### Transcript chunking and merging

Dataverse limits each record in `conversationtranscripts` to approximately 1 MB. Long or complex conversations are split across multiple records. The Agent Debugger:

1. Fetches all records whose `name` field contains the conversation ID
2. Groups records by session start time (to separate multiple sessions in the same conversation)
3. Within each session, sorts records by `metadata.BatchId` to restore chronological order
4. Concatenates all activity arrays into a single unified transcript

### Activity parsing

The merged transcript is then parsed event-by-event. The parser maintains a rolling buffer of debug state (pending steps, knowledge sources, token data) and flushes it onto each assistant message turn as it is encountered. The following trace event types are processed:

| Event | What is extracted |
|---|---|
| `message` | User/bot turns, adaptive card attachments, suggested actions |
| `DynamicPlanReceived` | Plan steps, parent–child plan nesting |
| `DynamicPlanStepTriggered` | Step start, LLM thought, step type |
| `DynamicPlanStepBindUpdate` | Step arguments (inputs) |
| `DynamicPlanStepFinished` | Step result (observation), execution time, state |
| `IntentRecognition` | Matched intent / topic name |
| `GenerativeAnswersSupportData` | Token counts, model name |
| `UniversalSearchToolTraceData` | Knowledge sources searched and returned |
| `KnowledgeTraceData` | Knowledge sources cited in the answer |
| `CodeTraceData` | Code execution output |
| `ConnectedAgentInitializeTraceData` | Connected agent session start |
| `ConnectedAgentCompletedTraceData` | Connected agent session end |
| `DynamicServerInitialize` / `DynamicServerToolsList` | MCP server info and tools |
| `ErrorTraceData` | Error codes and messages |
| `SessionInfo` | Overall session outcome |

---

## Troubleshooting

### Agent does not appear in the Environment or Agent dropdown

**Cause:** The agent has not been synced to Agent Inventory, or its `cat_istranscriptavailablecode` flag is not set.

**Resolution:**
1. Run a manual Agent Inventory sync for the environment in question.
2. Verify the agent record exists in the `cat_agentdetails` table in Dataverse.
3. Check that the `cat_istranscriptavailablecode` column is set to `1` on that record. The sync sets this flag when at least one transcript exists.
4. See [AGENT\_INVENTORY.md](https://github.com/microsoft/Power-CAT-Copilot-Studio-Kit/blob/main/AGENT_INVENTORY.md) for full sync instructions.

---

### Conversation ID not found in the dropdown

**Cause:** The conversation may be older than the 50-conversation default limit shown in the dropdown, or the transcript has not been written to Dataverse yet.

**Resolution:**
1. Type the conversation ID directly into the Conversation ID field — this triggers a full search across all transcripts for that agent (up to 100,000 records).
2. If the conversation just ended, wait 1–5 minutes for the transcript to be written to Dataverse, then refresh.
3. Verify the bot ID on the `cat_agentdetails` record matches the agent that ran the conversation.

---

### "Analyze" loads but shows no steps in the Debug Information panel

**Cause:** The transcript exists but contains only message-type activities with no diagnostic trace events. This typically happens when the conversation came from a channel that does not emit trace data (e.g., certain custom channels or very old schema versions).

**Resolution:**
1. Click the **View JSON** link in the Conversation Preview header to confirm activities are present.
2. Look for `type: "trace"` or `type: "event"` entries. If absent, trace logging may be disabled for this channel.
3. In Copilot Studio, verify that **Transcript logging** is enabled in the agent's **Settings → Advanced**.

---

### Connected Agent card shows "Load" but fetching fails

**Cause:** The Dataverse connection does not have read access to the child agent's transcript in the target environment, or the child transcript was written to a different environment.

**Resolution:**
1. Verify the connected agent's environment. The child conversation transcript is stored in the same Dataverse environment as the child bot.
2. If the child agent is in a different environment than the parent, the Dataverse connection must have read access to `conversationtranscripts` in that remote environment.
3. Check that the identity used by the connector has the **CSK - Administrator** role (or equivalent read rights on `conversationtranscripts`) in the child agent's environment.

---

### Steps show 0s duration in the Debug Information panel

**Cause:** Execution time is read from `DynamicPlanStepFinished.executionTime`. If the step finished events are missing or malformed, durations fall back to timestamp-based estimates.

**Resolution:**
1. Click the **View JSON** link in the Conversation Preview header and search for `DynamicPlanStepFinished`. Confirm the events are present and contain an `executionTime` field.
2. If events are missing, the conversation may have ended abruptly (e.g., `SystemError` outcome). Steps that never finished will show 0s.

---

### "Access denied" or blank page on load

**Cause:** The signed-in user does not have the **CSK - Administrator** or **System Administrator** role in the environment.

**Resolution:**
1. Ask an environment admin to assign the **CSK - Administrator** security role to the user in the target Dataverse environment.
2. If the app itself is in a different environment from the agent being debugged, roles must be granted in *both* environments.

---

### Transcripts appear incomplete (missing early messages)

**Cause:** Long conversations can span many 1 MB Dataverse records. If some records are missing or were purged, the merged transcript will have gaps.

**Resolution:**
1. Click the **View JSON** link in the Conversation Preview header and check the `mergedSessionKey` field on activities — gaps between batch IDs indicate missing chunks.
2. Dataverse transcript retention policies may have purged older records. Check the transcript retention setting in your environment's Copilot Studio configuration.
3. If retention is the issue, consider reducing conversation length or increasing the retention window.

---

### Steps show generic names (e.g., "unknown_12345") instead of readable topic names

**Cause:** The `botcomponents` table lookup failed or the component record was deleted.

**Resolution:**
1. Verify the user and connector have read access to the `botcomponents` table in the target environment.
2. If the component was deleted from Copilot Studio, the schema name has no matching record and the debugger falls back to the raw ID. This is expected for deleted topics or actions.
