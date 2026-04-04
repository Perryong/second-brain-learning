---
title: "OSCP Certification Sprint"
created: "2026-04-04"
type: project
status: active
priority: critical
tags:
  - project
  - oscp
  - cybersecurity
  - certification
deadline: "2026-05-15"
---

# OSCP Certification Sprint

> **Outcome:** Pass the OSCP exam in mid-May 2026 — score 70+ points, compromise and document 3-4 machines within 24 hours, submit a professional penetration testing report within 24 hours post-exam, and receive OSCP certification from Offensive Security.

**Exam Date:** Mid-May 2026 (~6 weeks away)
**Study Window:** 8:30pm – 10:30pm | 6 sessions/week
**Current Phase:** Week 1 — Environment setup + easy boxes

---

## Readiness Markers (Pre-Exam Checklist)

- [ ] 30+ vulnerable machines rooted and documented
- [ ] Enumeration speed: any target under 30 minutes (port scan → service ID → vuln research)
- [ ] Buffer overflow exploitation mastered (~25 guaranteed exam points)
- [ ] 10+ Linux privilege escalation techniques in muscle memory
- [ ] 8+ Windows privilege escalation techniques ready to deploy
- [ ] 3 full exam simulations completed (5 machines, 24-hour limit each)
- [ ] Professional pentest report template prepared and practiced

---

## High Priority / Critical

- [ ] Enroll in OSCP course / activate PWK lab access (if not done yet)
- [ ] Set up Kali Linux VM and test VPN connection to PWK lab environment
- [ ] Root first easy OSCP-style machine from TJNull list (start with "Lame" on HackTheBox)
- [ ] Complete PWK course modules 1-3 (introduction, information gathering, vulnerability scanning)

---

## Next Actions / Current Tasks

### Week 1 Milestones
- [ ] Root 3+ easy boxes from TJNull list and document each one
- [ ] Set up documentation system (Practice Log template, screenshot workflow)
- [ ] Join/verify access to r/oscp, OffSec Discord, and study group communities

### Week 2-3: Core Skills
- [ ] Work through buffer overflow module (PWK) — this is worth ~25 exam points
- [ ] Complete enumeration methodology: Nmap → Gobuster → Nikto → manual review
- [ ] Root 10+ machines total (mix of Linux and Windows)

### Week 4-5: Privilege Escalation Deep Dive
- [ ] Linux PrivEsc: SUID, sudo misconfigs, cron jobs, writable paths, kernel exploits
- [ ] Windows PrivEsc: SeImpersonatePrivilege, unquoted service paths, registry keys, AlwaysInstallElevated
- [ ] Reference: [LinPEAS/WinPEAS](https://github.com/carlospolop/PEASS-ng) | [GTFOBins](https://gtfobins.github.io/) | [LOLBAS](https://lolbas-project.github.io/)

### Week 6: Exam Simulations
- [ ] Exam simulation 1: 5 machines, 24-hour timer, full report
- [ ] Exam simulation 2: adjust weak areas from sim 1
- [ ] Exam simulation 3: final confidence check

---

## Key Resources

### GitHub
- [TJNull's OSCP Prep List](https://github.com/tjnull/OSCP-Stuff)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [PEASS-ng (LinPEAS/WinPEAS)](https://github.com/carlospolop/PEASS-ng)
- [SecLists](https://github.com/danielmiessler/SecLists)
- [Exploit-DB](https://www.exploit-db.com/)

### YouTube
- [IppSec](https://www.youtube.com/@ippsec) — HackTheBox walkthroughs (watch AFTER attempting)
- [TCM Security](https://www.youtube.com/@TCMSecurityAcademy) — practical PrivEsc courses
- [John Hammond](https://www.youtube.com/@_JohnHammond) — CTF and exploitation walkthroughs

### Platforms
- [HackTheBox](https://www.hackthebox.com/) — primary practice (TJNull list)
- [Proving Grounds](https://www.offensive-security.com/labs/) — OffSec-curated boxes
- [HackTricks](https://book.hacktricks.xyz/) — methodology wiki

---

## Someday/Maybe

- [ ] After OSCP: set up home lab with Vulnhub machines for continued practice
- [ ] Explore OSEP (OffSec Experienced Penetration Tester) after bug bounty phase

---

## Waiting On

*(Nothing currently blocked)*

---

## Machine Progress Log

| # | Machine | Platform | OS | Difficulty | Status | Date | Key Technique |
|---|---------|----------|----|------------|--------|------|---------------|
| 1 | | | | | | | |

*Target: 30+ machines before exam*

---

## Completed

*(Tasks will move here as done)*

---

## Links

- [[Assisting-User-Context]] — Perry's goals and context
- [[Post-OSCP-Bug-Bounty-Plan]] *(create after passing exam)*
