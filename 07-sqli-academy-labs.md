# SQL Injection Academy Labs: 6 Labs, 4 Oracles

PortSwigger Web Security Academy, SQL injection topic, solved October 2026 as training. These are public training labs, not client work and not real targets. No per-instance values, no secrets, no screenshots: only the patterns and the methodology.

This write-up is a lab companion to my general manual SQL injection methodology, already published as 04-manual-sqli-methodology.md. It does not repeat that guide: no detection-order theory, no second-order, no NoSQL, GraphQL, or ORM contexts. It covers only what these six Academy labs added: one filter bypass and five blind extractions, each through a different oracle. Two out-of-band labs are parked, not solved: they need Burp Collaborator, which is Pro-only, and I run Community.

## The point

Six labs, four oracles. The extraction logic never changes: ask a true/false question, read the answer, move to the next character. Only the channel changes: a filter, a content difference, an error code, a delay, or a visible error message.

## Pattern 1: Filters check one representation, the query sees another

Lesson: encoding layers decode before the query runs. A filter that checks only the raw input misses the decoded form. The fix is parameterized queries, not a longer blocklist.

Evidence: an XML-encoded UNION bypassed a keyword filter, using a single-column UNION once the filter was passed.

## Pattern 2: Boolean oracles, content vs error

Lesson: when the application swallows errors, compare responses. When it shows errors, compare status codes. Both are booleans, and either one reads the whole database.

Evidence: blind extraction through conditional responses, where the true and false cases returned different content; and blind extraction through conditional errors on Oracle, where the true case returned 200 and the false case returned 500.

## Pattern 3: Time is an oracle, and it scales to extraction

Lesson: a consistent delay on the true condition and none on the false condition confirms injection even when the page never changes. The same delay then extracts data character by character.

Evidence: pg_sleep(10) as a first-payload detection, confirmed on the first try; then time-delay extraction done by hand in small batches, verifying each character against the delay.

## Pattern 4: Visible errors leak data, but parameters have limits

Lesson: a database error that echoes query output is a direct channel, faster than any blind oracle. But every parameter has a length limit, so payloads must fit inside it.

Evidence: a PostgreSQL CAST to int turned a string into an error message displaying the value; and a tracking parameter truncated at 60 characters, which forced every payload to fit inside that limit.

## Reading responses: the oracle table

- **Content difference between true and false cases:** boolean oracle, extract through it.
- **500 vs 200 on the same input with one condition flipped:** error oracle, extract through it.
- **Consistent multi-second delay on true, none on false:** time oracle. Repeat to rule out network noise before extracting.
- **Visible database error containing query output:** direct leak, read it and shorten the next payload if needed.
- **Truncated or malformed behavior:** a length limit. Shorten the payload instead of debugging the injection.

## What I check first now

1. Confirm with a time delay before building anything.
2. Identify the oracle type: content, error, time, or visible error.
3. Extract in small verifiable batches, one character at a time.
4. Watch parameter length limits early, not after a failed extraction.
5. Out-of-band only when boolean and time channels are closed. That lab stays parked until the tooling question is reopened.

## Closing

Blind SQL injection is not guessing. It is reading through a keyhole, one boolean at a time, and the discipline is in verifying each answer before moving on. The full methodology, second-order cases, and non-classic contexts live in 04-manual-sqli-methodology.md. These six labs are the practice set that sits next to it.
