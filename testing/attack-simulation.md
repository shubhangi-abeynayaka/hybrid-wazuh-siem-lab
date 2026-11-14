# Attack Simulation & Validation Runbook

This document describes how test telemetry was generated to validate custom detection rules and the Active Response configuration in this lab. All simulations were performed from the Windows 11 host against the Ubuntu virtual machine.

| Scenario | Rule tested | MITRE ATT&CK | Result |
| --- | --- | --- | --- |
| 1. SSH brute force + automated containment | `100101` + `firewall-drop` (60s timeout) | T1110 (Brute Force) | Detected, IP temporarily blocked |
| 2. Windows Guest account enabled | `100200` | T1078 (Valid Accounts) | Validated via Discover |

---

## Lab Context

- **Wazuh manager:** Ubuntu VM in Oracle VirtualBox (NAT mode with port forwarding)
- **Endpoint:** Windows 11 host with the Wazuh agent installed (agent 002)
- **Key networking detail:** Traffic from the Windows host reaches the VM through the NAT gateway (`10.0.2.2`), which triggers Active Response isolation on the gateway interface (see [lessons-learned.md](../docs/lessons-learned.md))

---

## Before Testing: Validate the Rule Syntax

After editing `local_rules.xml` and updating `ossec.conf` with the 60-second timeout (`<timeout>60</timeout>`), check the rules before relying on them:

```bash
# Interactive rule tester: paste a raw log line and see which rules match
sudo /var/ossec/bin/wazuh-logtest

# Apply changes after editing configurations
sudo systemctl restart wazuh-manager
```

Sample line to paste into `wazuh-logtest` (paste it three times in a row to see the frequency rule trigger):

```text
Nov 14 15:36:06 ubuntu-vm sshd-session[11897]: Failed password for subhangi from 10.0.2.2 port 49932 ssh2
```

Keep these two logs open in separate terminals while testing:

```bash
sudo tail -f /var/ossec/logs/ossec.log
sudo tail -f /var/ossec/logs/active-responses.log
```

---

## Scenario 1: SSH Brute Force and Automated Containment

### Objective
Confirm that correlation rule `100101` fires after repeated SSH failures from one source IP, and that Active Response (`firewall-drop` configured with a 60-second timeout) blocks that IP temporarily.

### Execution
From PowerShell on the Windows host, attempted SSH logins with incorrect passwords through the VirtualBox port-forward (host port `2222` to guest port `22`) within a 120-second window:

```powershell
ssh subhangi@localhost -p 2222
# Enter an incorrect password at each prompt
```

### Expected Behavior
1. Each failed password matches baseline rule `5760`.
2. After 3 matching failures from the same source IP within 120 seconds, rule `100101` fires at level 10 (MITRE T1110).
3. Active Response runs `firewall-drop` against the source IP (`10.0.2.2`) with an automated 60-second expiration timeout.

### Observed Results
- **Detection:** Rule `100101` fired with `srcip` `10.0.2.2` and `frequency` 3, recorded in `active-responses.log` and the dashboard's SSH Authentication Activity panel.
- **Response:** `active-responses.log` recorded the `firewall-drop` script executing the containment command:

```text
2026/11/14 21:06:07 active-response/bin/firewall-drop: Starting
2026/11/14 21:06:07 active-response/bin/firewall-drop: {"version":1,"origin":{"name":"node01",...},"command":"add",...,"rule":{"level":10,"description":"Multiple SSH login failures observed from the same source IP","id":"100101",...}}
2026/11/14 21:06:07 active-response/bin/firewall-drop: Ended
```

- **Side effect & Timeout Tuning:** Because `10.0.2.2` is the NAT gateway, blocking it temporarily cut the SSH session and web dashboard access. However, thanks to the `<timeout>60</timeout>` configuration, the `iptables` drop rule automatically expired after 60 seconds, restoring connectivity without requiring manual hypervisor console intervention.

### Recovery (Emergency Fallback)
If testing requires an immediate manual reset, open the VM directly from the VirtualBox console and flush the rules:

```bash
sudo iptables -F
```

### Evidence
- `screenshots/03-active-response-firewall-drop.png`
- `screenshots/01-dashboard-overview.png`

---

## Scenario 2: Windows Guest Account Enabled

### Objective
Confirm that rule `100200` detects the built-in Guest account being enabled.

### Execution
From an elevated Command Prompt on the Windows host:

```cmd
net user guest /active:yes
```

Revert the change afterwards:

```cmd
net user guest /active:no
```

### Expected Behavior
1. The Wazuh agent forwards Windows Security event `4722` ("A user account was enabled").
2. Rule `100200` matches: parent rule `60103`, event ID `4722`, target user `Guest`.
3. A level 12 alert appears, tagged MITRE T1078.

### Observed Results
> **Status:** Validated in the OpenSearch Discover dashboard under rule ID `100200` with MITRE ATT&CK mapping T1078.

---

## Limitations
- Attacks are simulated from the same host running the lab, so there is no real attacker diversity.
- Thresholds (3 attempts in 120 seconds, 60-second block timeout) are chosen for rapid lab testing iteration speed rather than production security baselines.
- Only password-based SSH failures are covered; alternative brute-force methods (such as key-based or distributed attempts) are out of scope.\# Attack Simulation \& Validation Runbook



This document describes how I generated test telemetry to validate the custom detection rules and the Active Response configuration in this lab. All "attacks" are simulated from my own Windows 11 host against my own Ubuntu VM.



| Scenario | Rule tested | MITRE ATT\&CK | Result |

| --- | --- | --- | --- |

| 1. SSH brute force + automated containment | `100101` + `firewall-drop` | T1110 (Brute Force) | Detected, IP blocked |

| 2. Windows Guest account enabled | `100200` | T1078 (Valid Accounts) | See status note in Scenario 2 |



\---



\## Lab Context



\- \*\*Wazuh manager:\*\* Ubuntu VM in Oracle VirtualBox (NAT mode with port forwarding)

\- \*\*Endpoint:\*\* Windows 11 host with the Wazuh agent installed (agent 002)

\- \*\*Key networking detail:\*\* traffic from the Windows host reaches the VM through the NAT gateway, so the manager sees the source as `10.0.2.2` (see \[lessons-learned.md](../docs/lessons-learned.md))



\---



\## Before Testing: Validate the Rule Syntax



After editing `local\_rules.xml`, check the rules before relying on them:



```bash

\# Interactive rule tester: paste a raw log line and see which rules match

sudo /var/ossec/bin/wazuh-logtest



\# Apply changes

sudo systemctl restart wazuh-manager

```



Sample line to paste into `wazuh-logtest` (paste it three times in a row to see the frequency rule trigger):



```text

Nov 14 15:36:06 ubuntu-vm sshd-session\[11897]: Failed password for subhangi from 10.0.2.2 port 49932 ssh2

```



Keep these two logs open in separate terminals while testing:



```bash

sudo tail -f /var/ossec/logs/ossec.log

sudo tail -f /var/ossec/logs/active-responses.log

```



\---



\## Scenario 1: SSH Brute Force and Automated Containment



\### Objective



Confirm that correlation rule `100101` fires after repeated SSH failures from one source IP, and that Active Response (`firewall-drop`) blocks that IP.



\### Execution



From PowerShell on the Windows host, I attempted SSH logins with wrong passwords through the VirtualBox port-forward (host port `2222` to guest port `22`). Each attempt was an `ssh` command with several incorrect password prompts, all within a 120-second window:



```powershell

ssh subhangi@localhost -p 2222

\# Enter an incorrect password at each prompt

```



\### Expected Behavior



1\. Each failed password matches baseline rule `5760`.

2\. After 3 matching failures from the same source IP within 120 seconds, rule `100101` fires at level 10 (MITRE T1110).

3\. Active Response runs `firewall-drop` against the source IP (`10.0.2.2`).



\### Observed Results



\- \*\*Detection:\*\* rule `100101` fired with `srcip` `10.0.2.2` and `frequency` 3, as shown in `active-responses.log` and the dashboard's SSH Authentication Activity panel.

\- \*\*Response:\*\* `active-responses.log` recorded the `firewall-drop` script starting, receiving the alert for rule `100101`, and adding the block:



```text

2026/11/14 21:06:07 active-response/bin/firewall-drop: Starting

2026/11/14 21:06:07 active-response/bin/firewall-drop: {"version":1,"origin":{"name":"node01",...},"command":"add",...,"rule":{"level":10,"description":"Multiple SSH login failures observed from the same source IP","id":"100101",...}}

2026/11/14 21:06:07 active-response/bin/firewall-drop: Ended

```



\- \*\*Side effect:\*\* because `10.0.2.2` is the NAT gateway, blocking it also cut my SSH session and the dashboard port forward. Details and fixes are in \[lessons-learned.md](../docs/lessons-learned.md).



\### Recovery



If the lab locks you out, open the VM directly from the VirtualBox console and flush the rules:



```bash

sudo iptables -F

```



\### Evidence



\- `screenshots/03-active-response-firewall-drop.png`

\- `screenshots/01-dashboard-overview.png`



\---



\## Scenario 2: Windows Guest Account Enabled



\### Objective



Confirm that rule `100200` detects the built-in Guest account being enabled.



\### Execution



From an elevated Command Prompt on the Windows host:



```cmd

net user guest /active:yes

```



Revert the change afterwards:



```cmd

net user guest /active:no

```



\### Expected Behavior



1\. The Wazuh agent forwards Windows Security event `4722` ("A user account was enabled").

2\. Rule `100200` matches: parent rule `60103`, event ID `4722`, target user `Guest`.

3\. A level 12 alert appears, tagged MITRE T1078.



\### Observed Results



> \*\*Status:\*\* update this section after running the test. Record the alert's timestamp, rule ID, and level as seen in Discover (filter: `rule.id: 100200`), and save a screenshot as `screenshots/04-guest-account-alert.png`. If you did not run this test, say so here.



\---



\## Limitations



\- Attacks are simulated from the same host that runs the lab, so there is no real attacker diversity.

\- Thresholds (3 attempts in 120 seconds, 60-second block timeout) are chosen for quick lab testing, not tuned against real traffic.

\- Only password-based SSH failures are covered; other brute-force methods (for example, key-based or distributed attempts) are not tested.



