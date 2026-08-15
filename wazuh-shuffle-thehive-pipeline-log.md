# Wazuh → Shuffle → TheHive SOAR Pipeline
### Troubleshooting & Setup Log — August 12, 2026

> All IPs, hostnames, passwords, API keys, and webhook/tunnel URLs below have been redacted or replaced with placeholders for public GitHub publication. Replace placeholders with your own environment's values when reproducing this build.

---

## 1. Environment Overview

- **Wazuh VM** (Amazon Linux OVA, VMware Workstation)
  - IP: `<WAZUH_VM_IP>`
  - SSH: `ssh <wazuh-user>@<WAZUH_VM_IP>`

- **TheHive 4** running in Docker on the Wazuh VM (host networking mode)
  - Port: `9000`
  - Compose file: `~/thehive-lab/docker-compose.yml`
  - Both services (`thehive` + `elasticsearch`) have `restart: unless-stopped`
  - ⚠️ This install only has the old hyphenated Docker Compose binary. Use `docker-compose`, **not** `docker compose` (no space). Example: `sudo docker-compose ps`

- **Elasticsearch for TheHive** runs on port `9201` (`9200` is already taken by Wazuh's own internal indexer)

- **ngrok** exposes TheHive publicly at a permanent URL:
  - `https://<YOUR-NGROK-SUBDOMAIN>.ngrok-free.dev`
  - Runs as systemd service `ngrok.service`, auto-starts on VM boot, points at port 9000
  - Check status: `sudo systemctl status ngrok`
  - Health check: `curl -s https://<YOUR-NGROK-SUBDOMAIN>.ngrok-free.dev/api/status` (should return TheHive's version JSON)

- **Shuffle workflow**: `Wazuh-TheHive-Pipeline` (shuffler.io)
  - Structure: Webhook trigger node → `TheHive 1` HTTP POST node
  - Webhook URL (used in Wazuh's `ossec.conf`): `https://shuffler.io/api/v1/hooks/webhook_<REDACTED>`

- **TheHive organization**: `SOC-Lab`
  - `admin@thehive.local` — org-admin role
  - `shuffle-bot@soc-lab.local` — analyst role (used for day-to-day alert viewing)

- **Windows 10 target VM**
  - IP: `<WINDOWS_TARGET_IP>`, Wazuh agent ID `002`, agent name `WIN10`

- **Kali attacker VM**
  - IP: `<KALI_ATTACKER_IP>`
  - Attack tool: `crackmapexec` (SMB brute force against port 445)

- Wazuh's `<integration>` block (in `/var/ossec/etc/ossec.conf`) sends any alert with level ≥ 7 to the Shuffle webhook in JSON format:

```xml
<ossec_config>
  <integration>
    <name>shuffle</name>
    <hook_url>https://shuffler.io/api/v1/hooks/webhook_<REDACTED></hook_url>
    <level>7</level>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

---

## 2. Starting Symptom

A real SMB brute-force attack from Kali against Windows 10 was detected correctly by Wazuh (rule `60204`, level 10, "Multiple Windows Logon Failures" — confirmed in `/var/ossec/logs/alerts/alerts.log`). But nothing appeared automatically in Shuffle's execution history, even though:

- `wazuh-integratord` process was confirmed running (`ps aux`)
- A manual `curl POST` from the Wazuh VM directly to the Shuffle webhook URL succeeded (`{"success": true, ...}`)

So outbound connectivity was fine, but the automatic integration wasn't completing end-to-end.

---

## 3. Problem 1 — TheHive/Elasticsearch containers weren't running

**Diagnosis steps:**

1. Checked `ossec.conf`'s `<integration>` block — correct, properly nested, `<alert_format>json</alert_format>` present
2. Checked `<jsonout_output>` setting — `yes`, correct
3. Checked `/var/ossec/logs/alerts/` — files existed, being written
4. Checked `/var/ossec/logs/integrations.log` (integratord-specific send log, separate from `ossec.log`):
   ```
   sudo tail -n 50 /var/ossec/logs/integrations.log
   ```
   → Showed many `/tmp/shuffle-*.alert` entries being sent — proved Wazuh **was** sending alerts out
5. Checked `wazuh-integratord` process — confirmed running
6. Checked `ossec.log` for integratord startup messages — confirmed `"Started"` and `"Enabling integration for: 'shuffle'"`

**Conclusion:** Problem was not on the Wazuh side. Moved to Shuffle's Executions/Debug tab.

7. In shuffler.io → workflow → **Debug** tab (not Build — that's the editor view). Debug shows execution history with Refresh Runs and filters (All / Finished / Executing / Aborted).
8. Found multiple executions **were** happening (teal webhook icon = real automatic trigger vs. orange = manual Test Action). Every execution showed a red error icon.
9. Clicked into an execution:
   - The `$exec` data block showed real Wazuh alert data arriving correctly
   - The `TheHive 1` node (`post_create_alert`) showed:
     ```
     "status": 502
     "body": "Traffic successfully made it to the ngrok agent, but the agent failed to establish a connection to the..."
     ```

**Diagnosis:** Status 502 + that ngrok error means ngrok itself was reachable, but TheHive was not listening on port 9000 behind it.

**Fix:**

```bash
cd ~/thehive-lab
sudo docker-compose ps
```
→ Showed the table header but **zero containers** — confirmed nothing was running. (`docker compose ps` with a space fails on this install — `docker: 'compose' is not a docker command`.)

```bash
sudo systemctl status docker        # confirmed daemon healthy
sudo docker-compose up -d           # Container elasticsearch Started / Container thehive Started
sleep 30 && sudo docker-compose ps  # confirmed both "Up", no crash-loop
sudo systemctl status ngrok         # confirmed tunnel pointed at port 9000
curl -s https://<YOUR-NGROK-SUBDOMAIN>.ngrok-free.dev/api/status   # returned valid TheHive version JSON
```

**Result:** TheHive was back online. This fixed the 502, but a new error appeared next (see Problem 2).

> **Root lesson:** `restart: unless-stopped` only restarts containers that were running when Docker itself restarts — it does not survive a full VM power-cycle if the containers weren't up before shutdown.

---

## 4. Side Issue — Wazuh dashboard not loading

Two separate occasions came up:

**a) False alarm:** Dashboard actually was loading fine — the correct URL is `https://<WAZUH_VM_IP>` (HTTPS, self-signed cert, browser will warn — click Advanced → Proceed).

**b) Real issue — "Wazuh dashboard server is not ready yet":** Root cause was `wazuh-indexer` timing out on startup. Diagnosis:

```bash
sudo systemctl status wazuh-dashboard
# Active/running, but journal spammed ConnectionError every few seconds

sudo journalctl -u wazuh-dashboard -n 30 --no-pager | grep -i error
# [ConnectionError]: connect ECONNREFUSED 127.0.0.1:9200

sudo systemctl status wazuh-indexer
# Active: failed (Result: timeout)

free -h
# Available memory was fine (~5.1Gi) — not outright memory exhaustion

sysctl vm.max_map_count
# 262144 — already correctly set, not the cause

sudo tail -100 /var/log/wazuh-indexer/wazuh-cluster.log
# Log showed the JVM still mid-initialization (stuck after "Clustername: wazuh-cluster")
# when systemd's default startup timeout killed it
```

**Fix:** The indexer just needed more time to start, especially while competing for RAM with TheHive's Docker containers on an 8GB VM.

```bash
# Free up RAM by stopping TheHive containers during indexer startup
cd ~/thehive-lab && sudo docker-compose stop

# Extend systemd's startup timeout for the indexer
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d
sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf << 'EOF'
[Service]
TimeoutStartSec=300
EOF

sudo systemctl daemon-reload
sudo systemctl start wazuh-indexer
# Wait 2-3 minutes, then:
sudo systemctl status wazuh-indexer
```

> **Note:** `systemctl edit wazuh-indexer` (the interactive nano-based approach) failed with *"temporary file is empty"* when nothing was typed before saving — writing the override file directly with `tee` avoided that issue entirely.

Once the indexer was healthy, TheHive containers were started back up (`sudo docker-compose up -d`) and the dashboard loaded normally.

---

## 5. Side Issue — Windows clock skew scare (false alarm, but be careful)

While investigating why new alerts weren't showing up, Windows' clock (`echo %date% %time%`) was compared to the Wazuh server's clock (`date`, UTC). This led to an **incorrect** conclusion of a 1-day skew, and Windows' date was changed forward by a day.

After checking `tzutil /g` for the actual timezone offset, the original Windows clock was confirmed correct all along — the "skew" was just an unaccounted-for timezone difference.

**Fix:** Reverted the date change.

**Lesson:** Always check `tzutil /g` before comparing a local Windows clock to a UTC-based Linux server clock — never assume both are in the same timezone.

**Side effect:** Changing the Windows system date rotated the Wazuh agent's local `ossec.log`. Fixed with:
```
net stop WazuhSvc
net start WazuhSvc
```

---

## 6. Problem 2 (the main issue) — Broken JSON template in Shuffle's HTTP node

After TheHive was back up, a fresh test showed a new error:

```
"status": 400
"body": { "type": "BadRequest", "message": "Invalid Json: Unrecognized token '$': was expecting ..." }
```

**Root cause:** The `TheHive 1` node's Body field was still using Shuffle's generic default template for TheHive's "Create alert" action — full of `${description}`, `${flag}`, `${pap}` etc. that were never wired to real Wazuh data. The actual incoming data arrives as an `$exec` object (`$exec.title`, `$exec.severity`, `$exec.rule_id`, `$exec.id`, `$exec.timestamp`, and a nested `$exec.all_fields`).

**Debugging process (documented so the mistakes aren't repeated):**

| Attempt | What was tried | Result |
|---|---|---|
| 1 | `"title": "${$exec.title}"` | Still broke — never wrap `$exec.field` in `${ }` braces; that syntax is only for Shuffle's own internal variables |
| 2 | `"severity": {{ $exec.severity }}` (bare double-brace, no filter) | String fields worked, but numeric `severity` broke: `Invalid format for severity: FString({{ 3 }}), expected int` — Shuffle left literal `{{ }}` characters in for unfiltered double-brace fields |
| 3 | `{{ $exec.title \| default: "Wazuh Alert" }}` | Broke JSON — nested double-quotes inside a Liquid filter, inside an already-quoted JSON string, corrupts the JSON |
| 4 | Valid JSON but missing fields | TheHive rejected with `[Attribute date is missing]` / `[Attribute sourceRef is missing]` — these are mandatory |
| 5 | `"date": {{ "$exec.timestamp" \| date: "%s" }}000` | **Worked** — this line was correct from the start and never changed |
| 6 | Fully static/hardcoded body, no variables | Status 201 — proved the API/auth/schema were all fine, isolating the problem to the templating syntax specifically |
| 7 | Added fields back one at a time | Confirmed: `severity` must use **bare** `$exec.severity`, no braces at all — same rule as every other field |

---

## 7. Final, Confirmed Working Body

Saved in Shuffle's `TheHive 1` node (Build → click node → Setup tab → "Go to configuration" → Advanced tab → Body). Confirmed working via manual Test Action (201) **and** multiple real, live, fully-automatic brute-force attacks end-to-end:

```json
{
  "title": "$exec.title",
  "description": "Suspicious activity detected",
  "severity": $exec.severity,
  "pap": 2,
  "status": "New",
  "summary": "Detailed summary of the event",
  "type": "Internal",
  "source": "wazuh",
  "sourceRef": "$exec.id",
  "tags": ["wazuh"],
  "date": {{ "$exec.timestamp" | date: "%s" }}000
}
```

**Rules learned:**

1. For plain substitution, use **bare** `$exec.fieldname` — no braces at all, whether it's a JSON string or number.
2. Only use `{{ double braces }}` when applying an actual Liquid **filter** (the `|` pipe syntax) — e.g. date formatting.
3. Never nest double-quotes inside a Liquid filter that's itself inside a JSON double-quoted string.
4. TheHive's Create Alert API requires at minimum: `title`, `description`, `type`, `source`, `sourceRef`, `status`, `date`.
5. `date` must be epoch **milliseconds** (13 digits), not seconds and not an ISO string.
6. `sourceRef` must be unique per alert or TheHive rejects it as a duplicate — `$exec.id` (Wazuh's own alert ID) works well.
7. Shuffle's **Test Action** button runs live against whatever is currently typed, saved or not. But real live webhook-triggered runs use whatever was last actually **Saved** — always click Save once a Body is confirmed working.

---

## 8. Known Remaining Gap / Next Task

The `description` field is currently a static string ("Suspicious activity detected") for every alert — TheHive alone doesn't show which IP attacked or which account/host was targeted.

**Planned fix:** Pull richer detail from `$exec.all_fields`, which contains the full original Windows Event Log data, including fields like:

```
$exec.all_fields.win.eventdata.ipAddress       (attacker source IP)
$exec.all_fields.win.eventdata.targetUserName  (targeted account)
$exec.all_fields.win.eventdata.logonType
$exec.all_fields.win.system.computer           (target hostname)
```

**Next step:** Update just the `description` line, e.g.:

```json
"description": "Failed logon attempt from $exec.all_fields.win.eventdata.ipAddress targeting $exec.all_fields.win.eventdata.targetUserName"
```

A small, low-risk edit to an already-working template — not a rebuild. Test via Test Action first, then Save, then confirm with one real live attack. Exact field paths should be re-verified against a fresh Shuffle debug execution before finalizing, since the nesting was seen in the UI but not copied verbatim into this log.

---

## 9. Other Next Steps Discussed

1. **Clean up test/debug alerts** in TheHive from mid-session debugging (e.g. `sourceRef: test-ref-001`, or alerts with literal broken `{{ }}` in the title).
2. **Promote an alert into a Case** and practice the investigation workflow:
   - Open the alert preview (eye icon) → "Import alert as" → **Yes, Import**
   - Add Observables (attacker IP, target account, hostname) inside the Case
   - Add investigation Tasks, notes, and close with a verdict (True Positive / False Positive)
3. **This pipeline is generic** — any Wazuh alert reaching level 7+ automatically flows through the same path (Wazuh → integratord → Shuffle → TheHive), regardless of attack type. No reconfiguration needed for future attack types (SQLi, port scans, malware execution, etc.) as long as a matching Wazuh rule fires at level 7+.

---

## 10. Quick "Day 2" Startup Checklist

Almost everything here is permanently saved (ossec.conf, Shuffle's Body, ngrok's systemd service, Docker's restart policy). A fresh boot should mostly "just work" — but check:

1. Boot Wazuh VM, SSH in.
2. Check TheHive containers:
   ```bash
   cd ~/thehive-lab && sudo docker-compose ps
   # If not "Up" for both services:
   sudo docker-compose up -d   # wait ~30-60s for Elasticsearch
   ```
3. Check ngrok/TheHive reachability:
   ```bash
   curl -s https://<YOUR-NGROK-SUBDOMAIN>.ngrok-free.dev/api/status
   # 502 or connection error → repeat step 2
   ```
4. If the Wazuh dashboard shows "not ready yet," check `wazuh-indexer` status — see Section 4 for the timeout fix.
5. Boot target/attacker VMs as normal.
6. If alerts stop flowing, first suspect clock skew — check `date` (Wazuh, UTC) vs. `echo %date% %time%` + `tzutil /g` (Windows) — see Section 5.
7. Rule `60204` needs **8 distinct failed logons within 240 seconds** from the same source IP (`MS_FREQ=8` in `/var/ossec/ruleset/rules/0580-win-security_rules.xml`). Firing attempts too fast can cause Windows to collapse near-simultaneous `4625` events into fewer logged events than sent. Space attempts ~1 second apart:
   ```bash
   for i in wrongpass1 wrongpass2 wrongpass3 wrongpass4 wrongpass5 wrongpass6 wrongpass7 wrongpass8; do crackmapexec smb <TARGET_IP> -u <username> -p "$i"; sleep 1; done
   ```
   After ~8-9 attempts, Windows account lockout may trigger (also fires Wazuh rule `60115`). Unlock with:
   ```
   net user <username> /active:yes
   ```
