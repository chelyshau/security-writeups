# Broken Access Control: 12 Academy Labs, 5 Patterns

PortSwigger Web Security Academy, Access Control topic, solved October 2026 as training. These are public training labs, not client work and not real targets. No per-instance values, no secrets, no screenshots: only the patterns and the methodology.

## The point

Twelve labs look like twelve different bugs. They collapse into five patterns. Learn the pattern and you test faster, because you stop hunting for a specific bug and start checking whether a whole class of control is missing. The set: 8 apprentice labs and 4 practitioner labs.

## Pattern 1: Functions with no check at all

Lesson: every sensitive function needs a server-side role check on every request. Hiding the URL is not a control. If the check is missing, the function is public, whatever the path looks like.

Evidence: an admin panel that opened with no login at all, its path advertised in `/robots.txt`; and a panel with an "unpredictable" URL that was sitting in plain text in a JavaScript source file.

## Pattern 2: Role signals the client controls

Lesson: roles live server-side, in the session. Never trust role data from a cookie, a hidden field, or a request body. And whitelist the fields a mass-assignment endpoint accepts, because the client will send more than the form shows.

Evidence: login set an `Admin=false` cookie, and flipping it to `true` opened the panel; a profile-update request accepted a `roleid` field never meant to be user-editable, and the clue was the app echoing the full user object on a normal email change. The same class showed up in an API: a checkout response leaked a discount field set to 0, and a hand-built checkout request setting it to 100 was accepted.

## Pattern 3: Object references with no ownership check

Lesson: check ownership on every object access, not just on the front door. And never render a secret into a page, even inside a password field: the DOM is fully visible to whoever views source.

Evidence: changing the `id` parameter returned another user's page with their API key; a GUID used as the only protection leaked in a blog author link; a 302 to login still carried the other user's page and key in its body (browsers hide it, Burp shows it); an admin page pre-filled the current password into a password input (dots on screen, value in source).

## Pattern 4: Checks that cover only part of the flow

Lesson: authorize every step independently. A check on step one means nothing if step two trusts the caller, and a check on one method means nothing if the same action answers to another method.

Evidence: a role upgrade checked roles for POST only (401 as a low-privilege user); a nonsense method returned 400 "missing parameter", proving the check was tied to the method name, and the same request as GET, parameters moved to the query string, went through. In a two-step upgrade, step one checked roles (401, expected) while the confirmation step (`confirmed=true`) checked nothing, so replaying just the confirmation with a low-privilege session promoted the account.

## Pattern 5: Trust in client-controlled metadata

Lesson: Referer and rewrite headers are attacker-controlled input. Front-end path blocks are not security. The back-end must authorize the real resource, and it must never let a client header rewrite routing past an access check.

Evidence: an upgrade endpoint trusted the Referer header instead of the session role, and the captured admin request already carried a Referer pointing at the admin panel, so replaying it with a low-privilege session worked (a direct request without Referer returned unauthorized, the diagnostic that proves the check is header-based). A front-end blocked `/admin` outright (403), but the back-end honored `X-Original-URL`: a request to `/` with `X-Original-URL: /admin` opened the panel, and `/?username=carlos` with `X-Original-URL: /admin/delete` deleted the user.

## Reading responses: the diagnostics I reuse

Status codes tell you which layer said no, and that tells you what to try next:

- **401** means a role check fired. Ask what the control actually covers: this method, this step, this header?
- **400 "missing parameter"** after a method change means there is no role check on the new method, but the parameters are in the wrong place. For GET, they belong in the query string, not the body.
- **302 to the admin panel** means the action went through. Verify the new role, do not assume it.
- **A 401 response that issues a fresh session cookie** means the session you sent was not recognized at all. Do not debug with a dead session: take a fresh cookie from a live login on the same instance.

Two working habits from this set: copy values out of responses instead of retyping them from the screen, and always check what a redirect body contains, not just where it points.

## What I check first now

1. Request admin functions unauthenticated and as a low-privilege user.
2. Look for role data in cookies, hidden fields, and echoed API responses.
3. Swap object IDs, including GUIDs. Read redirect bodies, not just redirect targets.
4. Replay each step of a multi-step flow on its own. Vary the method.
5. On a 403, probe rewrite headers with an invalid path first. If the response changes, the header is processed. Only then point it at the target.

## Closing

None of these twelve is exotic. Each one is a check that exists somewhere but does not cover everywhere it needs to: not this function, not this field, not this object, not this method, not this step, not this header. That is the whole lesson, and it is a defensive one: access control is a property of every request, or it is not a property at all.
