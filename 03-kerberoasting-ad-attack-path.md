# Kerberoasting in Practice: From Service Tickets to Active Directory Compromise

*Lab environment: TryHackMe Active Directory rooms (PT1 coursework and PNPT exam preparation labs)*

## Overview

During Active Directory lab assessments, I repeatedly ran into the same situation: a low-privilege domain foothold, no obvious misconfiguration in front of me, and a domain controller that would not fall to anything noisy. In two separate lab engagements, the path forward turned out to be Kerberoasting — requesting Kerberos service tickets for domain service accounts and cracking them offline. This writeup walks through the technique as I actually applied it: why it works, how to find roastable accounts, how the cracking step fits into a longer attack chain, and what it looks like from the defensive side.

## Why Kerberoasting Works

In Active Directory, any service that authenticates via Kerberos registers a Service Principal Name (SPN) — something like `HTTP/webserver.corp.local` or `MSSQLSvc/dbserver.corp.local:1433`. When a domain user wants to talk to that service, the domain controller hands them a Ticket Granting Service (TGS) ticket, encrypted with the service account's password hash (usually RC4-HMAC).

The key insight: **any authenticated domain user can request a TGS for any SPN-bearing account, with no special privileges required.** The encryption is only as strong as the service account's password. And service account passwords are historically terrible — set once years ago, never rotated, often something like `Summer2021!` because nobody wants to break a production service by changing it.

So the attack is:
1. Authenticate as any low-privilege domain user.
2. Enumerate SPNs and request a TGS for each one.
3. Take the tickets offline and crack them with hashcat.
4. Log in as the service account — which frequently turns out to have local admin rights somewhere interesting.

No exploitation, no malware, no touching LSASS. Just asking the domain controller for tickets it is designed to hand out.

## Finding Kerberoastable Accounts

After establishing a foothold with domain credentials, the first step is SPN enumeration. From a Linux attack box, Impacket's `GetUserSPNs` does the LDAP query and the TGS requests in one pass:

```
impacket-GetUserSPNs <domain>/<user>:<password> -dc-ip <DC-IP> -request
```

The `-request` flag is what actually pulls the ticket material — without it, you only get the account list. In one lab this returned five SPN-bearing accounts; in another, just a single service account with an `HTTP` SPN. Quantity does not matter much — one weak password is enough.

What I look for at this stage:
- **How many SPNs exist.** More accounts means more chances that one has a crackable password.
- **What the SPNs suggest.** An `HTTP` or `MSSQLSvc` SPN hints at what the account can reach if compromised.
- **Whether the output looks "old."** In labs (and, frankly, in real estates) service accounts tend to accumulate over years without review.

One more thing: I do not request tickets for every SPN blindly. Bulk requests are noisy and trivially detectable, so experienced operators filter first — accounts with `adminCount=1`, members of privileged groups, or passwords untouched for years (`pwdLastSet`). This targeted approach — one high-value ticket requested quietly instead of fifty at once — is also what separates a careful assessment from a noisy one.

## Cracking the Tickets

The captured TGS hashes go to hashcat (mode 13100 for RC4-encrypted Kerberos 5 TGS):

```
hashcat -m 13100 tgs.txt rockyou.txt -r OneRuleToRuleThemAll.rule
```

In practice, this is where the attack usually succeeds or dies. A 25+ character random service password will not fall to any wordlist, and that is exactly the point — but in both of my lab runs, at least one ticket cracked within a reasonable wordlist-plus-rules pass. Service account passwords set by humans, years ago, under no rotation policy, are the norm rather than the exception.

One detail worth knowing: if a service account negotiates both RC4 and AES Kerberos encryption, the ticket request can be forced down to RC4 even when AES is available — and RC4-encrypted tickets (hashcat mode 13100) crack far more readily than AES ones (mode 19700). Always check which encryption types the account actually negotiates before assuming the stronger one will be used.

**Trade-off:** Kerberoasting is fully offline after the ticket request, so there is no sustained noisy activity — but the initial TGS requests are visible to the domain controller, and a mature Defender for Identity deployment will flag bulk ticket requests for SPNs. Requesting one or two tickets looks like normal user behavior; requesting fifty does not.

## What Comes Next

A cracked service account password is rarely the end of the chain — it is a pivot point. In my PT1 lab run, the cracked service account credential went straight into BloodHound analysis, which revealed a `GenericAll` permission over the Domain Admins group. One ACL edit later, my account was a domain admin. In the PNPT exam chain, cracked service credentials opened an SMB share that led, through several more hops, to a passback attack and full domain compromise.

The general pattern after a successful roast:
1. **Validate the credential** — confirm where the service account has rights (local admin on which hosts?).
2. **Map the attack path** — BloodHound or manual LDAP enumeration to find the shortest route from this account to Domain Admin.
3. **Move laterally** — WinRM, SMB, PsExec, or pass-the-hash depending on what the account permits.
4. **Document every hop** — in a real engagement, each step becomes a numbered finding with evidence.

Kerberoasting almost never gives you Domain Admin directly. What it gives you is a *better* set of credentials than the ones you started with, and in Active Directory, a better credential plus BloodHound is usually all it takes.

## Defensive Takeaways

Having run this attack from the offensive side, the defensive fixes are straightforward — the hard part is operational discipline, not technology:

- **Service account passwords must be long, random, and rotated.** 25+ characters generated by a password manager, rotated on a schedule. This single control defeats the entire attack.
- **Prefer AES Kerberos encryption and disable RC4 where possible.** AES-encrypted tickets (hashcat mode 19700) are significantly harder to crack than RC4 ones.
- **Migrate to Group Managed Service Accounts (gMSA) where possible.** gMSA passwords are long, random, rotated automatically by the domain controller every 30 days, and cannot be used for interactive logon — which removes the entire attack surface Kerberoasting depends on. This is the structural fix; everything above is mitigation.
- **Do not use regular user accounts as service accounts.** A service account should not be something someone also logs in with interactively.
- **Monitor TGS request patterns.** A single user requesting tickets for many different SPNs in a short window is anomalous and worth an alert in Defender for Identity or equivalent.
- **Audit SPNs regularly.** Every SPN is a potential roast target — if the service no longer exists, remove the SPN.

## Takeaways

Kerberoasting is one of those techniques that looks almost too simple to work — no exploit, no payload, just asking for tickets and cracking them at home. But it works precisely because it targets a human problem (bad service passwords, no rotation) rather than a software bug, and human problems do not get patched.

In both lab engagements where I used it, the roasted credential was the turning point of the assessment: the moment the engagement went from "stuck with a low-privilege shell" to "reading the domain's attack graph in BloodHound." If there is one Active Directory technique worth practicing until it is muscle memory, this is a strong candidate — the tooling is simple, the opsec cost is low, and the payoff is routinely a full domain compromise path.
