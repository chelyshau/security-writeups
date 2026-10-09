# Network Pivoting Techniques: A Practical Comparison

*Lab environment: TryHackMe "Wreath" network (internal pentest / red team simulation room)*

## Overview

During a multi-host internal network assessment, I needed to reach segments that
were not directly routable from my attack machine. The target network was
reachable only through a single compromised host acting as a pivot point, so
every tool had to run or tunnel through that foothold. I tested five different
pivoting approaches on the same scenario to compare their reliability and
overhead: SSH local/dynamic port forwarding, SOCKS proxies with proxychains,
Chisel, socat, and sshuttle.

## Scenario

- One externally reachable Linux host serves as the foothold.
- Several additional hosts (web server, Windows hosts) are only reachable
  from that foothold's internal interface.
- Goal: route scanning, HTTP, and RDP/WinRM traffic from my attack box through
  the foothold without installing extra software on the target hosts wherever
  possible.

## 1. SSH Local and Dynamic Port Forwarding

The simplest option when SSH access to the foothold is available. Local
forwarding maps a single remote port to a local one:

```
ssh -L <local_port>:<target_host>:<target_port> user@foothold
```

Dynamic forwarding (`-D`) turns the SSH connection into a SOCKS proxy, which
is more flexible when several internal hosts/ports need to be reached:

```
ssh -D <local_port> user@foothold
```

Paired with **proxychains** (or FoxyProxy in a browser), any tool can then be
routed through that SOCKS proxy without being pivot-aware itself. This was
my first choice for exploring the web application on the internal segment,
since it let me use Burp Suite and a browser against internal hosts with no
extra agent on the target.

**Trade-off:** reliable and built into OpenSSH, but one single, somewhat
fragile TCP session carries all the traffic, and some tools (raw ICMP scans,
certain UDP traffic) don't tunnel cleanly through a SOCKS proxy.

## 2. Metasploit `portfwd`

When the foothold was reached via a Meterpreter session rather than plain
SSH, `portfwd` creates a local TCP relay on the attack box:

```
portfwd add -l <local_port> -p <remote_port> -r <remote_host>
```

This is a per-port relay, not a full proxy, so it's best for forwarding one
or two known services (e.g., RDP or a specific web port) rather than general
network access. Metasploit's own docs note it's being superseded by the
`auxiliary/server/socks_proxy` module plus proxychains for exactly that
reason — I used the SOCKS module instead once I needed to reach more than
one host.

## 3. Chisel

A single static Go binary that creates an HTTP(S)-tunneled SOCKS proxy or
direct port forward between attacker and target, useful when the foothold
has no SSH server or egress is filtered to only allow outbound HTTP:

```
# attack box
./chisel server -p <port> --reverse
# foothold
./chisel client <attacker_ip>:<port> R:socks
```

Chisel was the most convenient option here because it needed no SSH daemon
on the foothold and tunneled cleanly over a single outbound HTTP connection,
which also made it a good stand-in for how a filtered or restrictive
network might only allow outbound web traffic.

## 4. socat

A general-purpose relay tool, good for one specific redirection without the
overhead of a full proxy:

```
socat TCP-LISTEN:<local_port>,fork TCP:<target_host>:<target_port>
```

I used socat to relay a single reverse-shell callback through the foothold
to a host that otherwise had no route back to my attack box — simple,
no dependencies beyond the binary itself, but it has to be set up
per-connection and doesn't proxy arbitrary traffic the way a SOCKS solution
does.

## 5. sshuttle

Essentially a transparent VPN built on top of SSH — no SOCKS configuration
or proxychains needed, since it rewrites the local routing table:

```
sshuttle -r user@foothold <internal_subnet>/24
```

Once running, any tool on the attack box (nmap, curl, browsers) could reach
the internal subnet directly, with no per-tool proxy configuration. This was
the fastest way to re-run a full `nmap` sweep against the internal range once
the foothold was established — much less setup friction than reconfiguring
proxychains for a scanning tool that doesn't support SOCKS natively.

## Comparison

| Tool | Needs install on target | Proxies arbitrary traffic | Best for |
|---|---|---|---|
| SSH `-D` + proxychains | No (SSH only) | Yes (SOCKS) | Ad hoc tool use, browser traffic |
| Metasploit `portfwd` | No (Meterpreter only) | No (single port) | One or two known services |
| Chisel | Yes (static binary) | Yes (SOCKS over HTTP) | Restrictive egress, no SSH |
| socat | Yes (binary) | No (single relay) | One-off shell/callback relay |
| sshuttle | No (SSH only) | Yes (full subnet) | Scanning, tools with no SOCKS support |

## Takeaways

- SSH-based options (dynamic forwarding, sshuttle) are the lowest-friction
  choice whenever SSH access to the pivot is already available — no extra
  binaries to transfer, and sshuttle in particular removes the need for
  proxy-aware tooling entirely.
- Chisel is the better fit once SSH isn't an option or the environment
  filters everything except outbound HTTP/HTTPS.
- For a single known service, a lightweight relay (`portfwd`, socat) is
  simpler to reason about than standing up a full proxy.
- In a real engagement I'd default to sshuttle or SSH dynamic forwarding
  first for speed, and keep Chisel in reserve for environments with tighter
  egress filtering.
