# Home SOC Lab

A personal Security Operations Center lab built to demonstrate
end-to-end SOC analyst skills: SIEM operations, detection
engineering, attack simulation, and incident documentation.

The lab uses Splunk as the SIEM, Sysmon for Windows endpoint
visibility, and Kali Linux as the attacker platform. All
attack techniques are mapped to MITRE ATT&CK and matched with
detection rules I wrote and tuned.

## Progress

- [x] **Milestone 1** — [Splunk setup on Ubuntu](./docs/01-splunk-setup.md)
- [x] **Milestone 2** — [Windows endpoint with Sysmon and Splunk Forwarder](./docs/02-windows-instrumentation.md)
- [ ] Milestone 3 — Attack simulation from Kali, first SPL detection
- [ ] Milestone 4 — Additional attack scenarios mapped to MITRE ATT&CK
- [ ] Milestone 5 — Investigation reports and final polish

## Architecture

- **SIEM:** Splunk Free on Ubuntu Server 24.04 LTS (4GB RAM, 2 cores, 80GB disk)
- **Endpoint (planned):** Windows 10 with Sysmon and Splunk Universal Forwarder
- **Attacker (planned):** Kali Linux for simulated reconnaissance, exploitation, and post-exploitation
- **Hypervisor:** VMware Workstation Pro 17
- **Networking:** NAT, all VMs isolated from the public internet

## About Me

Cybersecurity practitioner focused on SOC operations, vulnerability management, and offensive security as a defender's foundation. Currently working through TryHackMe SOC Level 1 in parallel with
this lab build.

**Certifications:** CompTIA PenTest+, INE eJPT, INE eWPT, OffSec Web Assessor, Blue Team Level 1, Splunk Core Certified User, CompTIA Security+, AWS Cloud Practitioner, Azure Fundamentals, CompTIA Cloud Essentials+, CompTIA ITF+.

**In progress:** Certified Red Team Professional (CRTP), OffSec Wireless Professional (OSWP).

[LinkedIn](https://linkedin.com/in/charles-neboh-128496204) ·
[TryHackMe Writeups](https://github.com/ARLOO0/tryhackme-writeups)
