# Phase 8: Active response engineering & automated threat containment

> Objective: Elevate the Wazuh deployment from a passive Security Information and Event Management (SIEM) system to an active Intrusion Prevention System (IPS). This phase focuses on configuring, deploying, and validating automated response scripts to immediately contain identified threats without requiring human intervention.

---

## 1. Overview

Phases 1–7 built a complete detection pipeline: deploy → monitor → detect → map → visualize. But detection alone doesn't stop an attacker — it only tells you something happened. Active response closes the loop by automatically executing containment actions the moment a detection rule fires, reducing the window between compromise and remediation from minutes (human triage) to seconds (automated response).

This phase engineers two distinct active response mechanisms: a custom broadcast alert for FIM-triggered persistence indicators, and a network-level IP ban for automated brute-force attacks. Both are mapped to custom detection rules from Phase 5 and validated through live attack simulation.

---

## 2. Component configuration & mapping

Two active response mechanisms were engineered and configured within the Manager's `ossec.conf`:

### 2.1 — Custom broadcast alert (FIM containment)

A custom Bash script (`alert-root.sh`) was deployed to the `Ubuntu_Admin` agent to execute a system-wide `wall` broadcast when triggered. This was mapped to **Rule 100001** (Critical modification of `/etc/passwd`).

The purpose is immediate operator notification — when an adversary modifies `/etc/passwd` to create a backdoor account, every active terminal on the endpoint receives an instant alert, ensuring the administrator is aware even if they are not monitoring the Wazuh dashboard at that moment.

### 2.2 — Network-level IP ban (brute-force containment)

The native Wazuh `firewall-drop` script was configured to interact directly with the agent's routing tables, automatically blocking the source IP of an attacker. This was mapped to **Rule 100002** (High-frequency SSH brute-force) with a strict **180-second timeout** to prevent permanent lockouts during testing.

When the brute-force correlation threshold is crossed (≥5 failures in 60 seconds from the same IP), the active response injects an `iptables` rule that drops all traffic from the attacker's IP, severing the attack in progress without any manual intervention.

---

## 3. Troubleshooting & architectural discoveries

During testing and validation, three environmental challenges were identified and resolved. These highlight the complexity of deploying automated security responses across containerized and virtualized environments (WSL2).

### Discovery 1 — FIM sensitivity & legitimate process triggers (false positives)

**Issue:** While attempting to create a secondary target user (`adduser dummy`), the custom active response script triggered immediately, halting the terminal.

**Analysis:** The Wazuh `syscheck` module detected the legitimate modification of `/etc/passwd` by the `adduser` binary in real-time. Rule 100001 fired as designed, and the active response executed the `wall` broadcast instantly.

**Takeaway:** This simultaneously proved the responsiveness of the detection-to-response pipeline and highlighted the necessity of rule tuning in production environments. Whitelisting legitimate administrative binaries (or scoping the active response to exclude known-good processes) would be required before deploying this configuration outside of a lab.

### Discovery 2 — Loopback whitelisting & IPv6 resolution

**Issue:** Initial brute-force simulations directed at `localhost` generated Rule 100002 alerts but failed to trigger the IP ban.

**Analysis:** Log analysis revealed two compounding factors. First, Windows PowerShell resolved `localhost` to the IPv6 address `::1`. Second, Wazuh's core engine contains a hardcoded `white_list` that inherently prevents active responses against loopback addresses (`127.0.0.1` / `::1`) to avoid self-denial-of-service.

**Resolution:** The attack vector was redirected to target the specific IPv4 address assigned to the WSL2 virtual network adapter (`hostname -I` for IP discovery), bypassing both the IPv6 resolution and the loopback whitelist.

### Discovery 3 — Minimal OS (WSL2 Ubuntu) dependency limitations

**Issue:** Despite targeting the correct IP, the IP ban still failed silently.

**Analysis:** Manual verification on the agent revealed the error `iptables: command not found`. WSL2 utilizes a "minimal" Ubuntu image optimized for fast boot times, which strips out standard network filtering subsystems. The `firewall-drop` script lacked the necessary OS dependencies to execute its payload.

**Resolution:** The `iptables` package was manually provisioned:

```bash
sudo apt install iptables -y
```

This effectively enabled network-level containment on the WSL2 endpoint.

---

## 4. Final validation & telemetry

Following the resolution of the architectural constraints, a final SSH brute-force attack was launched from the Windows host against the WSL2 Ubuntu agent using the correct network adapter IP.

**Offensive result:** The attacker's terminal experienced a hard network block, resulting in a `Connection timed out` state — confirming the `iptables` rule was injected and enforced in real-time.

**Defensive telemetry:** The OpenSearch Dashboards successfully correlated the complete event sequence:

| Timestamp | Agent | Description | Level | Rule ID |
|---|---|---|---|---|
| `Apr 8, 2026 @ 17:55:35` | Ubuntu_Admin | ALERT: High frequency SSH brute force attack | 10 | 100002 |
| `Apr 8, 2026 @ 17:55:37` | Ubuntu_Admin | Host Blocked by firewall-drop Active Response | 3 | 651 |

Rule 100002 fired first (detecting the brute-force pattern), followed two seconds later by Rule 651 (confirming the automated IP block was applied). This two-second window between detection and containment validates the effectiveness of the active response pipeline.

---

## 5. Key technical decisions

**Why `wall` broadcast for FIM containment instead of an automated remediation?** Automatically reverting `/etc/passwd` changes is dangerous — it could undo legitimate administrative actions or corrupt the file if the rollback logic is imprecise. A broadcast alert ensures the operator is immediately notified and can make an informed decision about remediation, which is the appropriate response for a persistence indicator in a lab environment where the context of each change matters.

**Why a 180-second timeout for the IP ban?** In a production environment, a permanent or long-duration ban would be appropriate. In this lab environment, where the attacker and target share the same WSL2 network and the "attacker" is the administrator's own machine, a 3-minute timeout provides sufficient validation window while preventing accidental permanent lockout during iterative testing.

**Why the troubleshooting process matters more than the final configuration.** The three discoveries (false positive triggers, loopback whitelisting, missing `iptables`) are not bugs — they are the exact type of environmental constraints that a SOC engineer encounters when deploying active response in real infrastructure. Documenting the diagnostic process demonstrates the analytical skills required to bridge the gap between "works in documentation" and "works in production."

---

*Previous: [Phase 7 — Custom SOC dashboard](./phase-7-custom-dashboard.md/) · Next: [Phase 9 — TheHive integration](./phase-9-thehive.md/)*
