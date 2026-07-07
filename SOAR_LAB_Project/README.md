# 🛡️ Automated SOAR Pipeline (n8n, FortiGate, Suricata)

## 📌 Project Overview
This project is a fully automated Security Orchestration, Automation, and Response (SOAR) pipeline deployed in a home lab environment. It provides a 1-click incident response mechanism that detects network intrusions, alerts analysts via a webhook, and dynamically isolates the threat by modifying Next-Generation Firewall (NGFW) policies via REST API.

## 🏗️ Architecture & Topology
* **Intrusion Detection (IDS):** Suricata
* **Orchestration (SOAR):** n8n
* **Enforcement (NGFW):** FortiGate VM
* **Alerting & UI:** Discord Webhooks
* **Hypervisor / Lab Environment:** EVE-NG

### Workflow Execution:
1. **Reconnaissance Detected:** Suricata detects an ICMP flood or Nmap OS probe from the Attacker VM.
2. **Alert Generation:** Suricata pushes a parsed log to a dedicated Discord SOC channel via webhook.
3. **Analyst Action:** The analyst reviews the payload and clicks the interactive **[BLOCK IP]** button.
4. **API Orchestration:** n8n intercepts the webhook, parses the target IP, and sends a `POST` request to the FortiGate API to create a new Address Object.
5. **State Commitment:** A strategic 5-second buffer allows the FortiGate disk I/O to commit the object.
6. **Threat Containment:** n8n sends a `PUT` request to update the `Quarantine_Group`, which is tied to a Sequence #1 Deny policy on the firewall, instantly dropping the attacker's traffic.

## ⚙️ Key Engineering Challenges Solved

### 1. Eradicating Alert Fatigue (Sensor Tuning)
Suricata's default behavior generated dozens of alerts per second during an ICMP flood, crashing the webhook pipeline. 
* **Solution:** Tuned the sensor at the rule level by injecting `threshold: type limit, track by_src, count 1, seconds 60;` directly into the `.rules` file, successfully throttling alerts to 1 per minute, per source IP.

### 2. Resolving API Race Conditions
Initial tests caused FortiGate database locks and 21-second connection timeouts because the orchestration engine fired the `POST` (create) and `PUT` (group) API calls simultaneously. 
* **Solution:** Implemented a mandatory 5-second wait state in the n8n logic flow, giving the FortiGate VM adequate time to commit the initial address object.

### 3. Overcoming Stateful Inspection Limitations
After successfully updating the firewall group, active attacker ping sessions continued to pass through the firewall.
* **Solution:** Re-architected the FortiGate policy table. Because firewalls do not kill established sessions by default, the automated `SOC_Auto_Block` deny rule was moved to absolute **Sequence #1**. Any new connection attempts post-block are instantly dropped.

## 🚀 How to Explore This Repo
* `/n8n-workflow/` - Contains the exported `.json` logic of the SOAR automation.
* `/suricata-config/` - Contains the custom `.rules` and `threshold.config` files used for rate-limiting.
* `/docs/` - Contains network topology diagrams and full project PDF documentation.
