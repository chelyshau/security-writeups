# Security Writeups

Hands-on penetration testing lab writeups covering network pivoting, internal
network attacks, web application exploitation methodology, and SQL injection
testing methodology.

**About me:** Independent penetration tester and security consultant with
10+ years of information security experience, including 6 years on the
defensive side (SIEM, DLP, vulnerability management, PCI DSS compliance) at
a bank, followed by hands-on offensive security work. Holds PNPT (Practical
Network Penetration Tester) and CEH certifications.

- LinkedIn: https://linkedin.com/in/ihar-chelyshau/
- TryHackMe: https://tryhackme.com/p/iharchelyshau

## Writeups

1. [Network Pivoting Techniques: A Practical Comparison](01-pivoting-techniques.md)
   - SSH forwarding, Metasploit portfwd, Chisel, socat, and sshuttle compared
   on the same internal network scenario.
2. [From Source Code Review to Remote Code Execution: A Git-Hosting Web App Case Study](02-git-server-code-review-rce.md)
   - exposed `.git` directory, manual PHP source review, command injection,
   and privilege escalation on a Windows host.
3. [Kerberoasting in Practice: From Service Tickets to Active Directory Compromise](03-kerberoasting-ad-attack-path.md)
   - requesting Kerberos service tickets, offline cracking with hashcat,
   and turning a low-privilege foothold into a domain compromise path.
4. [Beyond sqlmap: A Practical Methodology for Manual SQL Injection Testing](04-manual-sqli-methodology.md)
   - manual detection order (error-based, boolean-blind, time-based, out-of-band),
   second-order injection, NoSQL/GraphQL/ORM contexts, evidence standards,
   and remediation guidance.
5. [Broken Access Control: 12 Academy Labs, 5 Patterns](05-access-control-labs.md)
   - twelve Academy labs collapsing into five patterns: missing checks,
   client-controlled role signals, object references without ownership checks,
   partial-flow checks, and trust in client-controlled headers.
6. [Server-Side Request Forgery: 4 Academy Labs, 3 Patterns](06-ssrf-labs.md)
   - four Academy labs in three patterns: open fetch parameters, blacklist
   bypasses, and parser disagreement with open redirects as a bridge.
7. [SQL Injection Academy Labs: 6 Labs, 4 Oracles](07-sqli-academy-labs.md)
   - six Academy labs through four oracles: filter bypass, boolean content
   and error oracles, time delays, and visible errors.

All writeups are based on hands-on work in isolated training lab environments
(PortSwigger Web Security Academy and TryHackMe labs). No real client or
production data is included.
