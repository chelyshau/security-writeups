# Security Writeups

Hands-on penetration testing lab writeups covering network pivoting, internal
network attacks, and web application exploitation methodology.

**About me:** Independent penetration tester and security consultant with
10+ years of information security experience, including 6 years on the
defensive side (SIEM, DLP, vulnerability management, PCI DSS compliance) at
a bank, followed by hands-on offensive security work. Holds PNPT (Practical
Network Penetration Tester) and CEH certifications.

- LinkedIn: https://linkedin.com/in/ihar-chelyshau/
- TryHackMe: https://tryhackme.com/p/iharchelyshau

## Writeups

1. [Network Pivoting Techniques: A Practical Comparison](01-pivoting-techniques.md)
   — SSH forwarding, Metasploit portfwd, Chisel, socat, and sshuttle compared
   on the same internal network scenario.
2. [From Source Code Review to Remote Code Execution: A Git-Hosting Web App Case Study](02-git-server-code-review-rce.md)
   — exposed `.git` directory, manual PHP source review, command injection,
   and privilege escalation on a Windows host.
3. [Kerberoasting in Practice: From Service Tickets to Active Directory Compromise](03-kerberoasting-ad-attack-path.md)
   — requesting Kerberos service tickets, offline cracking with hashcat,
   and turning a low-privilege foothold into a domain compromise path.

All writeups are based on hands-on work in isolated training lab environments
(TryHackMe). No real client or production data is included.
