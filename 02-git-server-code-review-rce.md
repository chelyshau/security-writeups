# From Source Code Review to Remote Code Execution: A Git-Hosting Web App Case Study

*Lab environment: TryHackMe "Wreath" network (internal pentest / red team simulation room)*

## Overview

After pivoting into the internal network (see Part 1: Network Pivoting
Techniques), I found an internal host running a self-hosted Git server with
a PHP-based web front end. Rather than relying on exploit-db for a known CVE,
I pulled the application's own source and reviewed it directly — this
writeup walks through that process: enumeration, source acquisition, manual
code review, building a working remote-code-execution proof of concept, and
post-exploitation.

## Enumeration

A host-discovery sweep of the internal subnet (`nmap -sn`) turned up several
live hosts beyond the pivot box itself. A full TCP port scan against the
most interesting one showed HTTP, RDP, and WinRM open — a Windows host
serving a web application, which made it a reasonable next target: a
custom or lightly-maintained web app is usually a better entry point than
trying to attack RDP/WinRM directly.

## Source Acquisition

The web application exposed its `.git` directory publicly — a common
misconfiguration where a deployment process copies the whole project
directory, version control metadata included, straight into the webroot.
I used **GitTools** (specifically its `Extractor` component) to pull the
full repository history out of the exposed `.git` folder over HTTP, giving
me the application's complete PHP source rather than just what was visible
through the browser.

## Code Review

With the source in hand, I went through the PHP files looking for anywhere
user-controlled input reached a system call, file operation, or database
query without being sanitized. The application had a feature for running
server-side Git operations that was implemented by shelling out to the
system's `git` binary and interpolating a request parameter directly into
that command string — classic unsanitized command concatenation. Because
the parameter was never filtered for shell metacharacters, anything an
attacker could place in that field would execute in the context of the web
server process.

## Building the Exploit

I confirmed the vulnerability by sending a crafted HTTP POST request where
the vulnerable parameter contained a PowerShell one-liner instead of a
legitimate Git argument — a reverse-shell stager that opened a TCP
connection back to a listener on my attack box and piped commands through
it. The request executed successfully and returned a shell running as the
web service's own account.

## Privilege Confirmation & Impact

Running `whoami` through the resulting shell showed it was running as a
highly privileged local service account — not just "a shell on the box,"
but one with enough rights to create new local accounts directly. To
demonstrate real-world impact rather than stopping at code execution, I
created a new local user and added it to the local Administrators and
Remote Management Users groups, then used that account to establish a
stable interactive session (Evil-WinRM) instead of relying on the fragile
initial reverse shell.

## Post-Exploitation

From the stabilized session I pulled credential material from memory using
Mimikatz, run through an RDP session with a mapped drive, confirming that an
attacker landing through this single unsanitized parameter could move from
unauthenticated web access to full local compromise and credential theft in
a handful of steps.

## Root Cause & Remediation

- **Root cause:** user-supplied input passed directly into a shell command
  string instead of using a parameterized API or an allow-list of expected
  values.
- **Exposed `.git` directory:** deployment process copied version-control
  metadata into the public webroot, giving an attacker the full source and
  commit history for free.
- **Excessive service account privileges:** the web application's service
  account had local administrative rights it did not need for its function.

**Recommendations:**
1. Never build shell commands by string concatenation with user input — use
   language-level Git bindings or a strict argument allow-list instead.
2. Exclude `.git` (and other VCS metadata) from the web server's document
   root in the deployment pipeline, or block access to it at the web server
   config level.
3. Run the web application under a dedicated low-privilege service account,
   separate from any account with local admin rights.
4. Add file-integrity/command-execution monitoring on the host so an
   anomalous child process spawned by the web server (e.g., `powershell.exe`
   launched by a PHP worker) triggers an alert.

## Takeaways

This was a good example of why source-level review matters even when no
public CVE exists for an application: the exposed `.git` folder turned what
could have been a black-box guessing exercise into a direct read of the
vulnerable code path, which made both the exploit and the eventual written
report far more precise than "this endpoint behaves strangely with special
characters."
