# Orchestrator

The Orchestrator is an AI agent that interprets a natural-language security request, plans the tasks needed to fulfill it, runs those tasks, and returns a formatted response. It pauses to request approval before actions that need it.

## Release information

The following table lists the release details of the Orchestrator agent.

- **Version**: 1.0.0

- **Certified**: Yes

- **Publisher**: Fortinet

- **Scope**: AI Assistant bot

- **Verified with Models**: Fortinet FortiAI (AI model Medium)

## Installation

The Orchestrator is installed with the FortiAI solution pack. It needs no separate installation.

## Configuration and prerequisites

This section lists the prerequisites and describes how to configure the Orchestrator.

### Prerequisites

The Orchestrator requires the following.

- FortiSOAR v8.0.1 and later.
- The FortiAI solution pack.
- At least one LLM configuration. FortiSOAR ships the **High Reasoning** and **Low Reasoning** configurations, with **Low Reasoning** as the default.
- The **FortiSOAR Module Management** MCP server configuration.

### Configuring the Orchestrator

Use the following steps to open and configure the Orchestrator.

1. Navigate to **Content Hub** > **Discover**.
2. Select **AI Agent** under **Content Type** in the **Filters** pane, or type `orchestrator` in the search bar.
3. Select the **Orchestrator** card.
4. Select the **Configurations** tab.
5. Select a **Configuration Type**.
    - **Default System Configuration:** Uses the configuration that ships with FortiSOAR. No further input is needed.
    - **Custom Configuration:** Displays the fields described in the following table.
6. Select **Save**.

The following table describes the **Custom Configuration** fields.

| Field              | Required | Description                                                                                                                                  |
|--------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Configuration Name | Yes      | A name for this configuration.                                                                                                               |
| LLM Provider       | Yes      | The LLM configuration that sets the reasoning level. Higher reasoning gives more in-depth analysis. Lower reasoning gives faster responses for simpler tasks. |
| MCP Server         | Yes      | The MCP server configurations that give the Orchestrator access to tools, resources, and integrations. Select all four listed in [Prerequisites](#prerequisites). |
| Masking Agent      | No       | The agent that detects and masks IOCs before data is sent to the LLM. Defaults to **IOC Masking**.                                           |

## Execution flow

The Orchestrator is not invoked directly. The **AI Assistant bot** calls it in the background when a request needs several coordinated steps, such as finding related alerts and then blocking their IP addresses.

A request runs through the following stages.

1. **Planning:** The Orchestrator extracts and validates the entities in the request, retrieves a matching SOP from **Organization Context**, and derives a plan of tasks. If the request lacks information, for example the record to act on, it pauses and asks a question.

  The Orchestrator matches SOPs from **Organization Context** as follows.

  - Only SOPs in the `GENERIC_SOP` category are searched.
  - Up to five SOPs are retrieved, and an LLM selects the most appropriate one. Only selections with high or medium confidence are used.
  - If the request names an SOP, that SOP is retrieved directly, without scoring.

     The following agents are excluded when the Orchestrator agent creates and executes a plan:

    - Alert Investigation
    - Investigation Hypothesis Generation
    - Investigation Planner
    - Investigation Summarization
    - Task Planner
    - IOC Masking
    - Orchestrator
    - Chat Assistant
    - Connector Generation
    - Playbook Generator

2. **Execution:** The Orchestrator runs the tasks using other AI agents. Before an action that needs approval, it pauses and asks the analyst to approve or deny it.

3. **Synthesis:** The Orchestrator returns a response with a summary, next actions, and the evidence behind the result.

When the Orchestrator pauses, the analyst replies in the **AI Assistant bot**. The bot sends the reply with the same `request_id`, and the Orchestrator resumes the paused request.

The masking agent detects and masks IOCs before any data is sent to the LLM, and unmasks them in the response.

> [!Tip]
> To view each step the Orchestrator ran, select **Trace Flow** under the bot response in the **AI Assistant bot**.

## Limits

The Orchestrator applies the following fixed limits.

- A plan contains at most 15 tasks. A request that needs more ends with the `unplannable` status.
- A failing task is retried up to three times, then marked as failed.
- The Orchestrator asks for missing information in up to three clarification rounds. If the request is still incomplete, it ends with the `unplannable` status.
- If a reply does not answer a clarification question, the Orchestrator rephrases the question up to two times, then ends the request with the `unplannable` status.
- If a reply does not clearly approve or deny an action, the Orchestrator asks again up to two times, then denies the action.

## Input parameters

The **AI Assistant bot** sends the input as a JSON object with the following parameters.

| Parameter      | Required  | Type   | Description                                                                                                                                                       |
|----------------|-----------|--------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `raw_query`    | Yes       | string | The analyst's message. On a new request, this is the request. On a resume, this is the reply to the open clarification or approval.                              |
| `request_type` | No        | string | One of `playbook` or `other`. Selects which additional capabilities and feasibility checks apply. Default: `other`.                                               |
| `context`      | No        | object | Context that helps plan generation, such as `userId`, `pageName`, and `recordIRI`. Additional keys are accepted, for example the `record` object of the open page. |
| `request_id`   | On resume | string | The handle of a paused request. Omit it to start a new request. Include it to resume the paused request with that ID.                                            |

The following example starts a new request.

```json
{
  "raw_query": "Find related alerts and block the IP addresses",
  "request_type": "other",
  "context": {
    "userId": "3451141c-bac6-467c-8d72-85e0fab569ce",
    "pageName": "alerts",
    "recordIRI": "/api/3/alerts/7d2b1c4e-9f3a-4b8e-a1d6-5c0e2f8b9a31"
  }
}
```

The following example resumes a paused request.

```json
{
  "raw_query": "203.0.113.10",
  "request_id": "b6f1e0a2-3c4d-4e5f-8a9b-0c1d2e3f4a5b"
}
```

## Response

The Orchestrator returns a JSON object with the following parameters.

| Parameter       | Type   | Description                                                                                                       |
|-----------------|--------|-------------------------------------------------------------------------------------------------------------------|
| `request_id`    | string | The ID of the request. Use it to resume a paused request.                                                         |
| `status`        | string | The outcome of the request. See [Status values](#status-values).                                                  |
| `answer`      | string | The formatted response, in Markdown by default.                                                                   |
| `pending`       | object | The open question. Set only when `status` is `needs_clarification` or `awaiting_approval`. |
| `error`         | object | A machine-readable reason. Set only when `status` is `rejected` or `unplannable`. |
| `created_at`    | string | The time the response was created, in ISO 8601 format.                                                           |
| `actions_taken` | array  | The actions the Orchestrator performed.                                                                           |

### Status values

The `status` parameter returns one of the following values.

| Value                 | Description                                                                 |
|-----------------------|-----------------------------------------------------------------------------|
| `in_progress`         | The request is still running.                                               |
| `success`             | The request completed.                                                      |
| `success_no_action`   | The request completed, and no action was needed.                            |
| `partial`             | Some tasks completed and others failed.                                     |
| `failed`              | The request failed.                                                         |
| `denied`              | An action was denied, so the request did not complete.                      |
| `awaiting_approval`   | The request is paused until an action is approved or denied.                |
| `needs_clarification` | The request is paused until the analyst provides missing information.       |
| `rejected`            | The request or resume was refused, for example because of an unknown `request_id`. |
| `unplannable`         | The Orchestrator could not build a plan for the request.                    |

### Response examples

The following examples show a request that pauses for clarification, pauses for approval, and then completes.

> [!Note]
> These examples are illustrative. Values differ in your environment.

The following response asks the analyst which IP address to block.

```json
{
  "request_id": "b6f1e0a2-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
  "status": "needs_clarification",
  "answer": "The alert has two IP addresses. Which one do you want to block?",
  "pending": {
    "pause_id": "e4a7c2d9-1b3f-4c5e-9d8a-7f6b5e4d3c2b",
    "type": "clarification",
    "question": "Which IP address do you want to block?",
    "expected_type": "choice",
    "options": [
      {"label": "203.0.113.10", "value": "203.0.113.10", "kind": "choice"},
      {"label": "198.51.100.25", "value": "198.51.100.25", "kind": "choice"}
    ],
    "expires_at": "2026-10-09T15:42:10Z"
  },
  "error": null,
  "created_at": "2026-10-08T15:42:10Z",
  "actions_taken": []
}
```

The following response asks the analyst to approve the block.

```json
{
  "request_id": "b6f1e0a2-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
  "status": "awaiting_approval",
  "answer": "Blocking 203.0.113.10 changes the firewall configuration. Approve to continue.",
  "pending": {
    "pause_id": "a1c3e5f7-2b4d-4f6a-8c0e-1d3f5a7b9c2e",
    "type": "approval",
    "question": "Approve blocking 203.0.113.10?",
    "expected_type": "decision",
    "options": [
      {"label": "Approve", "value": "approve", "kind": "decision"},
      {"label": "Deny", "value": "deny", "kind": "decision"}
    ],
    "expires_at": "2026-10-09T15:43:02Z"
  },
  "error": null,
  "created_at": "2026-10-08T15:43:02Z",
  "actions_taken": []
}
```

The following response reports the completed request.

```json
{
  "request_id": "b6f1e0a2-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
  "status": "success",
  "answer": "Blocked 203.0.113.10 on the firewall.",
  "pending": null,
  "error": null,
  "created_at": "2026-10-08T15:43:40Z",
  "actions_taken": [
    {"action": "block_ip", "target": "203.0.113.10", "status": "success"}
  ]
}
```

The following response rejects a reply to a pause that was already answered.

```json
{
  "request_id": "b6f1e0a2-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
  "status": "rejected",
  "answer": "This request has already been resumed.",
  "pending": null,
  "error": {"code": "already_resumed", "detail": "This request has already been resumed."},
  "created_at": "2026-10-08T15:44:05Z",
  "actions_taken": []
}
```