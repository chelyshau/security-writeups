# Server-Side Request Forgery: 4 Academy Labs, 3 Patterns

PortSwigger Web Security Academy, SSRF topic, solved October 2026 as training. These are public training labs, not client work and not real targets. No per-instance values, no secrets, no screenshots: only the patterns and the methodology. One blind SSRF lab is parked, not solved: it needs Burp Collaborator, which is Pro-only, and I run Community.

## The point

Four labs look like four different tricks. They collapse into three patterns. SSRF is not about clever URLs. It is about a server that fetches whatever it is told, and a filter that checks the wrong thing.

## Pattern 1: A fetch parameter that takes any destination

Lesson: if a parameter reaches a server-side request, the attacker chooses the destination. The fix is to validate the final URL server-side and allowlist the permitted destinations. Client-side restrictions and parameter names are not controls.

Evidence: a stock-check parameter pointed at the local admin interface and reached it; the same parameter scanned an internal address range and found another back-end system that answered differently from the dead hosts.

## Pattern 2: Blacklists fail against alternate spellings

Lesson: every encoding layer is another spelling of the same destination. A filter that bans a list of spellings loses to the next spelling. Allowlist the destination instead of banning spellings.

Evidence: a localhost blacklist passed by 127.1; a filter that decoded only once passed by double-encoding, encoding the encoded form by hand.

## Pattern 3: URL parsers disagree, and open redirects bridge the gap

Lesson: a path the front-end treats as internal can carry an absolute URL the back-end fetches. Validate after the final parse, not before it, and never let an open-redirect parameter choose a fetch destination.

Evidence: a full external URL was rejected with a 400, but a relative path carrying an absolute internal URL passed and reached the internal admin delete function.

## Reading responses: the diagnostics I reuse

- **Dead vs live internal hosts:** dead hosts answer with an error status and a short body, the live one answers 200 with a longer body. The difference is the oracle for the range scan.
- **400 "Illegal character":** the URL is malformed. Check for stray characters, for example a trailing space.
- **400 "Invalid external stock check url":** a full external URL is blocked. Try a relative path carrying the URL instead.
- **200 on an internal path:** the fetch succeeded. Verify the action happened, do not assume it.

## What I check first now

1. Find parameters that fetch URLs: stock checks, image loads, document converters, webhooks.
2. Try localhost variants: 127.1, 0.0.0.0, and alternate spellings.
3. Try encoding layers: single, double, and mixed.
4. Scan internal ranges and read response differences, not just status codes.
5. Treat open-redirect parameters as SSRF carriers, not just redirect bugs.
6. Blind variants need an out-of-band channel. That lab stays parked until the tooling question is reopened.

## Closing

SSRF is a trust problem: the server trusts its own network position, and the attacker borrows it. The fix is boring in the best way: validate the final destination server-side, allowlist it, and never let user input choose where the server connects.
