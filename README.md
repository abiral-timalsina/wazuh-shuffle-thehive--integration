# Wazuh → Shuffle → TheHive SOAR Lab

End-to-end SOAR pipeline built in a home lab: **Wazuh** detects attacks, **Shuffle** orchestrates the response, and **TheHive** automatically receives and manages the resulting alerts — no manual steps in between.

This repo documents the full build, including the real debugging process (errors, wrong attempts, root causes, and fixes) rather than just a clean "it worked" writeup.

---

## Architecture

```
┌─────────────┐      brute-force       ┌──────────────┐
│    Kali     │ ─────────────────────► │  Windows 10  │
│  (attacker) │        (SMB/445)       │   (target)   │
└─────────────┘                        └──────┬───────┘
                                               │ Wazuh agent
                                               ▼
                                        ┌──────────────┐
                                        │    Wazuh     │
                                        │ (SIEM/rules) │
                                        └──────┬───────┘
                                               │ integratord (level 7+)
                                               ▼
                                        ┌──────────────┐
                                        │   Shuffle    │
                                        │ (SOAR/webhook)│
                                        └──────┬───────┘
                                               │ HTTP POST (via ngrok)
                                               ▼
                                        ┌──────────────┐
                                        │   TheHive    │
                                        │ (alert/case) │
                                        └──────────────┘
```

## What it does

1. A simulated attacker (Kali) runs an SMB brute-force against a Windows 10 target.
2. Wazuh detects the failed logon pattern (rule `60204`, "Multiple Windows Logon Failures") and flags it at severity level 10.
3. Wazuh's `integratord` automatically forwards any alert ≥ level 7 to a Shuffle webhook.
4. Shuffle receives the alert and posts it to TheHive's Create Alert API (exposed via an ngrok tunnel).
5. TheHive shows the alert to the analyst — ready to be investigated and promoted into a case.

All of this runs **automatically**, end-to-end, with no manual intervention once an attack occurs.

## Tools used

| Tool | Role |
|---|---|
| [Wazuh](https://wazuh.com/) | SIEM — log collection, detection rules, alerting |
| [Shuffle](https://shuffler.io/) | SOAR — workflow automation, webhook orchestration |
| [TheHive](https://thehive-project.org/) | Case management — alert triage and investigation |
| Docker | Runs TheHive + Elasticsearch |
| ngrok | Exposes local TheHive instance to Shuffle's cloud webhook |
| Kali Linux / crackmapexec | Attack simulation |

## Repo contents

- [`wazuh-shuffle-thehive-pipeline-log.md`](./wazuh-shuffle-thehive-pipeline-log.md) — full build & troubleshooting log: every error hit, root cause, and fix, including the exact working Shuffle JSON template and the reasoning behind it
- Screenshots (if included) showing Wazuh alerts, Shuffle execution history, and TheHive alerts

## Key things I learned

- How Wazuh's `<integration>` block and `integratord` forward alerts by severity level to an external webhook
- How to debug a SOAR pipeline layer by layer — isolating whether a failure is network, auth, schema, or templating
- Shuffle's Liquid templating quirks (`$exec.field` vs `{{ }}` double-brace syntax) and how JSON/Liquid interact
- TheHive's Create Alert API requirements (mandatory fields, epoch-millisecond dates, unique `sourceRef`)
- Real-world debugging habits: isolating variables with a fully static test payload before reintroducing dynamic fields one at a time
- Practical Windows/Wazuh detection tuning — how rule thresholds (e.g. `MS_FREQ`) affect whether an attack actually triggers a rule

## Status

✅ Fully working end-to-end (attack → detection → automated alert in TheHive)
🔜 Next: enrich alert descriptions with attacker IP / target account pulled from Wazuh's raw event data, and practice promoting alerts into full TheHive Cases with observables

---

*Built as part of my ongoing SOC analyst portfolio. Feedback welcome.*
