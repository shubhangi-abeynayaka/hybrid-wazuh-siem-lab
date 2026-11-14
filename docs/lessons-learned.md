# 🛠️ Lessons Learned & Technical Limitations

This document outlines the technical challenges, debugging workflows, and architectural limitations encountered during the development of this hybrid SIEM lab.

---

## 1. Active Response locked me out of my own lab

* **Problem:** When testing the automated `firewall-drop` active response script against simulated SSH brute-force attacks (Rule `100101`), Wazuh successfully blocked the attacking source IP, but immediately severed all management access to the environment.
* **Cause:** The lab infrastructure operated in VirtualBox NAT mode. Because of this network translation, incoming SSH traffic from the Windows host resolved on the Ubuntu guest as the hypervisor gateway (`10.0.2.2`). When Active Response executed `iptables` to drop `10.0.2.2`, it inadvertently blocked the entire host-to-guest bridge, dropping both the active SSH terminal session (port `2222`) and the OpenSearch web dashboard (port `4433`).
* **The 60-Second Timeout Twist:** Initially, with no explicit timeout defined in `ossec.conf`, Wazuh applied a permanent block, requiring emergency recovery by opening the local VirtualBox GUI console directly on the Ubuntu hypervisor guest as root and flushing the firewall rules (`sudo iptables -F`). To optimize testing iteration speed and prevent permanent lockouts, I added an explicit `<timeout>60</timeout>` tag to the active-response block, ensuring the `iptables` drop rule automatically expires after 60 seconds while still fully validating automated containment.
* **What I'd Change:**
  1. **Network Architecture:** Configure the VM using a Host-Only or Bridged adapter rather than NAT so that the host machine has a dedicated, non-gateway IP address.
  2. **Active Response Allowlisting:** Explicitly add administrative host IPs and gateway addresses to Wazuh’s global `<white_list>` configuration block to ensure security tools never target management interfaces.
  3. **Staged Rollout:** Test active response scripts using logging-only or non-critical interfaces before applying automated blocking rules to production firewalls.

---

## 2. NAT hides the real source IP

* **Problem:** Network-based detection and automated containment rely heavily on accurate attribution of source identifiers. 
* **Cause:** In VirtualBox NAT configurations, network address translation obscures the true host interface. 
* **Takeaway:** Recognizing that Ubuntu perceived incoming traffic from `10.0.2.2` rather than `127.0.0.1` was critical for ensuring that correlation rules (`<same_source_ip />`) and firewall rules targeted the correct network layer.

---

## 3. Validating rules before relying on them

* **Problem:** Modifying SIEM configuration files blindly and restarting services can lead to silent syntax failures or unparsed log streams.
* **Cause:** The Wazuh rules engine requires strict XML validation and relies on decoders to parse raw text into structured fields *before* rules evaluate them. (Note: Decoders handle field normalization; child rules built on parents like `60103` or `5760` simply inherit those pre-decoded fields and restrict evaluation to events the parent already matched).
* **Fix & Best Practice:** 
  * Always test custom XML rules using the built-in validation utility prior to restarting services:
    ```bash
    /var/ossec/bin/wazuh-logtest
    ```
  * Monitor service health and active response triggers directly via diagnostic logs rather than guessing:
    * `/var/ossec/logs/ossec.log`
    * `/var/ossec/logs/active-responses.log`
    * `/var/ossec/bin/wazuh-manager` status checks.

---

## 4. Limitations of this lab

To maintain technical accuracy, this lab operates under specific constraints:
* **Single-VM Architecture:** The SIEM manager and the target application run on a single host machine rather than a distributed enterprise network.
* **Simulated Attacker:** Attack simulation originates from the local host machine rather than an external, untrusted threat actor.
* **Threshold Tuning:** Brute-force frequency and timeframe parameters (`frequency="3" timeframe="120"`) are optimized for rapid lab testing rather than high-volume production baseline tuning.
