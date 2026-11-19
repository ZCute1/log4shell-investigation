# Investigating Log4Shell (Codepath's CYB 102 Capstone)

**Team:** Audrey Flores, Hoang Do, Eric Fisher, Saksham Tehri, Teniola Zuriel Olu-Silas

## 📄 Presentation Slides
👉 **[View Full Presentation Deck (PDF)](./CYB102_Presentation.pdf)**

---

## 🎯 Investigation Overview
Using network packet captures (PCAP) and Windows Security Auditing logs from the OTRF Security Dataset, our team investigated a suspected Log4Shell exploit against a Windows host (`WORKSTATION5`) running Apache Tomcat.

**Core Question:** *Did the Log4Shell request only hit the server, or did it actually lead to code execution?*

**Verdict:** **Confirmed Code Execution.** The vulnerable Tomcat/Log4j instance processed the malicious JNDI payload, initiated an outbound LDAP callback, and spawned an encoded PowerShell process that communicated with the attacker's C2 server.

---

## 🔍 Attack Timeline & Key Evidence

| Time (UTC) | Stage | Evidence Source | Key Finding |
| :--- | :--- | :--- | :--- |
| **08:54:05** | Initial Exploit | PCAP & Event `5156` | Attacker (`192.168.3.5`) sent an HTTP `GET` request to `192.168.2.5:8080` with a `${jndi:ldap://...}` payload in the `User-Agent` header. |
| **08:54:07** | LDAP Callback | PCAP & Event `5156` | `tomcat9.exe` initiated an outbound LDAP callback to `192.168.3.5:1389`. |
| **08:54:08** | Code Execution | Event `4688` | `tomcat9.exe` spawned `powershell.exe` running a hidden, Base64-encoded command (`-NonI -W Hidden -NoP -Exec Bypass -Enc`). |
| **08:54:09** | Post-Exploitation | Event `4663` & `5156` | PowerShell accessed LSA/Winlogon registry keys and established outbound HTTP traffic over port `443` to `192.168.3.5`. |

---

## 🛠️ Dataset, Tools & Playbooks Used
* **Dataset:** [OTRF Security Datasets — Log4Shell](https://securitydatasets.com/notebooks/compound/Log4Shell.html) (PCAP & Windows Security Auditing logs)
* **Wireshark:** Packet inspection and TCP stream reconstruction (`tcp.stream eq 1`).
* **Windows Security Auditing Logs:** Host-level process creation (`4688`), network connections (`5156`), and registry object access (`4663` / `4656`).
* **Incident Response Playbooks ([CERT Société Générale IRM](https://github.com/certsocietegenerale/IRM/tree/main/EN)):**
  * `IRM-5`: Malicious Network Behaviour
  * `IRM-2`: Windows Intrusion
---

## 🛡️ Remediation Summary
1. **Contain:** Isolate `WORKSTATION5` from the network.
2. **Preserve:** Retain PCAPs, Windows event logs, timestamps, and IOCs.
3. **Eradicate:** Patch or remove vulnerable Log4j libraries, terminate malicious PowerShell processes, and remove persistence artifacts.
4. **Recover:** Rotate credentials, rebuild if necessary, and safely restore services.
