# Autonomous SOC Triage Agent with Approval (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `soc-agent-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "Autonomous SOC Agent (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table soc_triage in Supabase SQL Editor\n3. Slack channel: soc-triage (change in the Slack node)\n4. Click Execute workflow: 4 mock security events, the AI Agent triages each one\nRules: risky actions or critical assets need human approval\nReal client use: replace Mock Events with a Webhook from your SIEM/EDR",
        "height": 260,
        "width": 580
      },
      "id": "8fc04ede-9a3d-4b45-b65c-d04f953e5b39",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "32728d86-b608-4473-8fe8-d570ff0d4e04",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { event_id: 'EV-001', source: 'firewall', host: 'web-prod-01', ip: '185.220.101.5', detail: '400 failed SSH logins in 2 minutes from one external IP' } },\n  { json: { event_id: 'EV-002', source: 'edr', host: 'hr-laptop-22', ip: '203.0.113.77', detail: 'A process is sending outbound beacons every 60 seconds to an unknown domain' } },\n  { json: { event_id: 'EV-003', source: 'identity', host: 'finance-db-01', ip: '198.51.100.23', detail: 'Admin login from two different countries within 10 minutes' } },\n  { json: { event_id: 'EV-004', source: 'scanner', host: 'dev-test-05', ip: '10.0.0.15', detail: 'Scheduled internal vulnerability scan completed' } }\n];"
      },
      "id": "d9b9f4d5-bcd8-477f-a965-8550b5e767e5",
      "name": "Mock Events",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "const assets = { 'web-prod-01': 'high', 'hr-laptop-22': 'medium', 'finance-db-01': 'critical', 'dev-test-05': 'low' };\nreturn $input.all().map((it) => ({ json: Object.assign({}, it.json, { asset_criticality: assets[it.json.host] || 'unknown' }) }));"
      },
      "id": "5e1a58e3-6384-4ab7-be53-3ed4f9a30996",
      "name": "Enrich Events",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Triage this security event.\nEvent: {{ $json.event_id }} from {{ $json.source }}\nHost: {{ $json.host }} (asset criticality: {{ $json.asset_criticality }})\nSource IP: {{ $json.ip }}\nDetail: {{ $json.detail }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a SOC triage analyst. Use the threat_intel_lookup tool with the source IP to check its reputation. Respond with ONLY a JSON object, no markdown, no extra text: {\"severity\":\"low|medium|high|critical\",\"verdict\":\"false_positive|suspicious|confirmed_threat\",\"action\":\"ignore|monitor|block_ip|isolate_host|reset_credentials\",\"summary\":\"one sentence\"}"
        }
      },
      "id": "418e565c-3061-47ac-be13-1c3010bfa5da",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        680,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "c8828b30-6e15-459b-be16-21f97e88967c",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        660,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "name": "threat_intel_lookup",
        "description": "Looks up the reputation of an IP address. Input: the IP address only, for example '185.220.101.5'. Returns reputation and notes as JSON.",
        "jsCode": "const intel = {\n  '185.220.101.5': { reputation: 'malicious', notes: 'Known Tor exit node used for brute-force attacks' },\n  '203.0.113.77': { reputation: 'suspicious', notes: 'Recently registered infrastructure linked to malware command servers' },\n  '198.51.100.23': { reputation: 'unknown', notes: 'No reports found' },\n  '10.0.0.15': { reputation: 'internal', notes: 'Internal scanner address' }\n};\nreturn JSON.stringify(intel[String(query).trim()] || { reputation: 'unknown', notes: 'No data' });"
      },
      "id": "27c583e1-8462-42f7-afa7-38f90aa47724",
      "name": "Threat Intel Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        880,
        420
      ]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Enrich Events').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst sev = ['low', 'medium', 'high', 'critical'].includes(String(r.severity).toLowerCase()) ? String(r.severity).toLowerCase() : 'medium';\nconst action = ['ignore', 'monitor', 'block_ip', 'isolate_host', 'reset_credentials'].includes(r.action) ? r.action : 'monitor';\nconst sensitive = ['isolate_host', 'reset_credentials'].includes(action);\nconst critical = ['critical', 'high'].includes(ctx.asset_criticality) && ['high', 'critical'].includes(sev);\nreturn { json: {\n  event_id: ctx.event_id,\n  source: ctx.source,\n  host: ctx.host,\n  severity: sev,\n  verdict: r.verdict || 'suspicious',\n  action,\n  asset_criticality: ctx.asset_criticality,\n  status: (sensitive || critical) ? 'needs_approval' : 'auto',\n  summary: String(r.summary || '').slice(0, 300)\n}};"
      },
      "id": "c3fc2dcf-b395-45af-95b7-4472d2b2af33",
      "name": "Parse Triage",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1040,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "629bf5dd-e6fc-49a8-9569-64f31421a671",
              "leftValue": "={{ $json.status }}",
              "rightValue": "needs_approval",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "beb82074-f051-488a-8125-0bc083fe1afd",
      "name": "Needs Approval?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1260,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "soc-triage",
          "mode": "name"
        },
        "text": "=:shield: *Approval needed: {{ $json.action }}*\nEvent: {{ $json.event_id }} on {{ $json.host }} ({{ $json.asset_criticality }})\nSeverity: {{ $json.severity }} | Verdict: {{ $json.verdict }}\n{{ $json.summary }}",
        "otherOptions": {}
      },
      "id": "95e00eba-ac0f-4044-b493-40f9177d3a1d",
      "name": "Slack Approval Request",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1520,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "soc_triage",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "event_id": "={{ $json.event_id }}",
            "source": "={{ $json.source }}",
            "host": "={{ $json.host }}",
            "severity": "={{ $json.severity }}",
            "verdict": "={{ $json.verdict }}",
            "action": "={{ $json.action }}",
            "asset_criticality": "={{ $json.asset_criticality }}",
            "status": "={{ $json.status }}",
            "summary": "={{ $json.summary }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "event_id",
              "displayName": "event_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "source",
              "displayName": "source",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "host",
              "displayName": "host",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "severity",
              "displayName": "severity",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "verdict",
              "displayName": "verdict",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "action",
              "displayName": "action",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "asset_criticality",
              "displayName": "asset_criticality",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "summary",
              "displayName": "summary",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "4be93acd-bbb1-4b93-9b5f-dcd5e0f05513",
      "name": "Save Triage",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1520,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Events",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Events": {
      "main": [
        [
          {
            "node": "Enrich Events",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Enrich Events": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Triage",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Triage": {
      "main": [
        [
          {
            "node": "Needs Approval?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Needs Approval?": {
      "main": [
        [
          {
            "node": "Slack Approval Request",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Triage",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Triage",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Threat Intel Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
