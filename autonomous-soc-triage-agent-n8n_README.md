# Autonomous SOC Triage Agent with Approval (n8n)

An n8n workflow where an AI Agent (Gemini) triages security events like a SOC analyst: it checks the source IP with a threat-intel tool, sets severity, verdict and an action, and a rules step decides whether the action needs human approval on Slack.

▶ **Demo video:** https://drive.google.com/file/d/1Hoc1lbLwMCEPf31qkbptyrq0q32MB9Y5/view

## How it works

1. Mock security events arrive (SSH brute force, beaconing process, impossible-travel admin login, internal scan)
2. Each event is enriched with the criticality of the host (low, medium, high, critical) from a lookup in code
3. The AI Agent calls the `threat_intel_lookup` tool with the source IP and returns severity, verdict, action and a one-sentence summary as JSON
4. A Code node validates the answer. Actions `isolate_host` and `reset_credentials`, and any high or critical severity on a high or critical host, are marked `needs_approval`
5. An IF node sends those to a Slack approval request, everything is saved to Postgres

```
Manual Trigger -> Mock Events -> Enrich Events -> AI Agent (Gemini + threat_intel_lookup tool)
-> Parse Triage -> Needs Approval?
   yes -> Slack Approval Request -> Save Triage
   no  -> Save Triage
```

## AI Agent

AI Agent node with one tool (`threat_intel_lookup`). The approval rule is plain code, so the agent can recommend a sensitive action but cannot decide on its own that it may run.

## Tech stack

n8n, AI Agent, Gemini, Slack, Postgres (Supabase)

## Setup

1. Copy the JSON from `workflow.md` into a file named `soc-agent-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `soc-triage`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: the mock inputs run through all branches

No other API keys are needed, the tool nodes are self-contained mock tools.

## Database

```sql
create table if not exists public.soc_triage (
  id bigint generated always as identity primary key,
  event_id text,
  source text,
  host text,
  severity text,
  verdict text,
  action text,
  asset_criticality text,
  status text,
  summary text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| EV-001 firewall: 400 failed SSH logins on web-prod-01 (high asset) | Likely block_ip, approval if severity is high or critical |
| EV-002 EDR: beaconing every 60s on hr-laptop-22 | Likely isolate_host, needs approval |
| EV-003 identity: admin login from two countries on finance-db-01 (critical asset) | Likely reset_credentials, needs approval |
| EV-004 scanner: scheduled internal scan on dev-test-05 (low asset) | Likely ignore, auto |

## Notes

- The approval step is a Slack message only. Nothing flows back from Slack yet, so no action is executed automatically
- The AI decides severity and action, so results can vary between runs. The approval rule does not
- Replace Mock Events with a Webhook from your SIEM or EDR and `threat_intel_lookup` with a real VirusTotal or AbuseIPDB call for client use
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
