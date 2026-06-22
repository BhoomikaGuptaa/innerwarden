# n8n Agent Guard Integration

This guide shows how to use InnerWarden Agent Guard with an n8n workflow. The workflow checks the current security context before an automation runs, validates a command before execution, and pauses or blocks risky automation when Agent Guard reports elevated risk.

## Overview

n8n workflows can trigger actions automatically, including HTTP requests, file operations, shell commands, and AI agent tool calls. When these workflows run on self-hosted infrastructure, unsafe commands or compromised automation can create security risk.

InnerWarden Agent Guard exposes local API endpoints that n8n can call before executing sensitive actions:

- `GET /api/agent/security-context` checks the current host threat level.
- `POST /api/agent/check-command` validates a command before execution.

The recommended workflow pattern is:

1. Trigger the n8n workflow.
2. Check InnerWarden security context.
3. Pause or stop if `threat_level` is `high` or `critical`.
4. Validate the command with Agent Guard.
5. Execute only if Agent Guard returns `allow`.
6. Pause for review or stop if Agent Guard returns `review` or `deny`.

## Prerequisites

- A running InnerWarden installation.
- Agent Guard enabled.
- A self-hosted n8n instance that can reach the InnerWarden Agent Guard API.
- Basic familiarity with n8n HTTP Request and IF nodes.

By default, the examples below assume InnerWarden is reachable at:

```text
http://localhost:3121
```

If n8n runs in Docker, `localhost` may refer to the n8n container instead of the host machine. In that case, use the correct host address for your environment, such as `host.docker.internal` where supported.

## Step 1: Check the Security Context

Add an **HTTP Request** node before the automation performs a sensitive action.

Recommended node name:

```text
Check Security Context
```

Configure the node:

| Field | Value |
| --- | --- |
| Method | `GET` |
| URL | `http://localhost:3121/api/agent/security-context` |
| Authentication | `None` |
| Response Format | `JSON` |

Example response:

```json
{
  "threat_level": "elevated",
  "active_incidents": 3,
  "blocked_ips_24h": 47,
  "recommendation": "Reduce external network calls. SSH brute-force campaign active.",
  "last_updated": "2026-04-18T14:32:00Z"
}
```

## Step 2: Pause Risky Automation When Threat Level Is High

Add an **IF** node after the security context request.

Recommended node name:

```text
Threat Level Safe?
```

Configure the IF node to continue only when the threat level is not high-risk.

Example condition:

```text
{{ $json.threat_level }}
```

Continue the workflow when the value is not one of:

```text
high
critical
```

If the threat level is `high` or `critical`, route the workflow to a notification, manual review, or stop node instead of continuing with execution.

Recommended behavior:

- `low` or `normal`: continue
- `elevated`: continue with caution or require review, depending on your policy
- `high` or `critical`: pause or stop automation

## Step 3: Validate a Command Before Execution

Add another **HTTP Request** node before the workflow executes a command.

Recommended node name:

```text
Check Command
```

Configure the node:

| Field | Value |
| --- | --- |
| Method | `POST` |
| URL | `http://localhost:3121/api/agent/check-command` |
| Authentication | `None` |
| Send Body | `JSON` |
| Content-Type | `application/json` |
| Response Format | `JSON` |

Example JSON body:

```json
{
  "command": "ls -la /var/log/",
  "agent_id": "n8n-workflow"
}
```

For a dynamic command from an earlier n8n node, use an expression:

```json
{
  "command": "{{ $json.command }}",
  "agent_id": "n8n-workflow"
}
```

Example allowed response:

```json
{
  "decision": "allow",
  "risk_score": 5,
  "matched_rules": [],
  "message": "Command appears safe"
}
```

Example denied response:

```json
{
  "decision": "deny",
  "risk_score": 100,
  "matched_rules": ["remote-code-execution-pipe"],
  "message": "Blocked: piped remote code execution"
}
```

## Step 4: Execute Only When Agent Guard Allows It

Add an **IF** node after the command validation request.

Recommended node name:

```text
Command Allowed?
```

Condition:

```text
{{ $json.decision }}
```

Continue only when the value equals:

```text
allow
```

Suggested routing:

| Agent Guard decision | n8n behavior |
| --- | --- |
| `allow` | Continue to the execution node |
| `review` | Pause and notify a human reviewer |
| `deny` | Stop the workflow and log the blocked command |

Do not execute a command when Agent Guard returns `review` or `deny`.

## Example Workflow

The example workflow uses this structure:

```text
Manual Trigger
  -> Set Example Command
  -> Check Security Context
  -> Threat Level Safe?
      -> false: Stop or Notify
      -> true:
          -> Check Command
          -> Command Allowed?
              -> false: Stop or Notify
              -> true: Execute Approved Action
```

This pattern checks both the system threat level and the command-level risk before allowing automation to continue.

## Example n8n Workflow JSON

The following JSON can be used as a starting point for an n8n workflow export. Review and adapt node names, URLs, and command execution behavior before using it in production.

```json
{
  "name": "InnerWarden Agent Guard Example",
  "nodes": [
    {
      "parameters": {},
      "id": "manual-trigger",
      "name": "Manual Trigger",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [240, 300]
    },
    {
      "parameters": {
        "values": {
          "string": [
            {
              "name": "command",
              "value": "ls -la /var/log/"
            }
          ]
        },
        "options": {}
      },
      "id": "set-example-command",
      "name": "Set Example Command",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [460, 300]
    },
    {
      "parameters": {
        "url": "http://localhost:3121/api/agent/security-context",
        "responseFormat": "json",
        "options": {}
      },
      "id": "check-security-context",
      "name": "Check Security Context",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [680, 300]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.threat_level }}",
              "operation": "notIn",
              "value2": "high,critical"
            }
          ]
        }
      },
      "id": "threat-level-safe",
      "name": "Threat Level Safe?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [900, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://localhost:3121/api/agent/check-command",
        "sendBody": true,
        "contentType": "json",
        "jsonBody": "={\"command\":\"{{$node['Set Example Command'].json['command']}}\",\"agent_id\":\"n8n-workflow\"}",
        "responseFormat": "json",
        "options": {}
      },
      "id": "check-command",
      "name": "Check Command",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1120, 220]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.decision }}",
              "operation": "equals",
              "value2": "allow"
            }
          ]
        }
      },
      "id": "command-allowed",
      "name": "Command Allowed?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [1340, 220]
    },
    {
      "parameters": {
        "values": {
          "string": [
            {
              "name": "status",
              "value": "Command approved by Agent Guard. Continue with execution."
            }
          ]
        },
        "options": {}
      },
      "id": "approved-action",
      "name": "Approved Action Placeholder",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [1560, 140]
    },
    {
      "parameters": {
        "values": {
          "string": [
            {
              "name": "status",
              "value": "Automation paused or blocked by Agent Guard."
            }
          ]
        },
        "options": {}
      },
      "id": "blocked-action",
      "name": "Blocked or Review Placeholder",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [1560, 320]
    }
  ],
  "connections": {
    "Manual Trigger": {
      "main": [[{"node": "Set Example Command", "type": "main", "index": 0}]]
    },
    "Set Example Command": {
      "main": [[{"node": "Check Security Context", "type": "main", "index": 0}]]
    },
    "Check Security Context": {
      "main": [[{"node": "Threat Level Safe?", "type": "main", "index": 0}]]
    },
    "Threat Level Safe?": {
      "main": [
        [{"node": "Check Command", "type": "main", "index": 0}],
        [{"node": "Blocked or Review Placeholder", "type": "main", "index": 0}]
      ]
    },
    "Check Command": {
      "main": [[{"node": "Command Allowed?", "type": "main", "index": 0}]]
    },
    "Command Allowed?": {
      "main": [
        [{"node": "Approved Action Placeholder", "type": "main", "index": 0}],
        [{"node": "Blocked or Review Placeholder", "type": "main", "index": 0}]
      ]
    }
  },
  "active": false,
  "settings": {},
  "versionId": "example",
  "meta": {
    "templateCredsSetupCompleted": true
  }
}
```

## Security Notes

- Always check before executing. Do not execute first and validate later.
- If Agent Guard is unreachable, fail closed and do not run the command.
- Use a dedicated `agent_id` such as `n8n-workflow` so Agent Guard logs are easy to trace.
- Do not expose the Agent Guard API publicly.
- Review `review` and `deny` responses before changing rules or workflow behavior.

## Troubleshooting

### n8n cannot reach `localhost:3121`

If n8n is running in Docker, `localhost` points to the container itself. Use the host address reachable from the container, such as:

```text
http://host.docker.internal:3121
```

or the correct Docker network hostname for your deployment.

### The workflow stops unexpectedly

Check the response from the `Check Security Context` and `Check Command` nodes. If `threat_level` is `high` or `critical`, or if `decision` is `review` or `deny`, the workflow should pause or stop by design.

### A safe command is marked for review

Inspect the `matched_rules` and `message` fields in the Agent Guard response. Update the workflow policy only after confirming that the command is expected and safe for your environment.