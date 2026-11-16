# API Testing Academy Labs: The Front End Is Not the Boundary

PortSwigger Web Security Academy, API testing topic, solved as training (labs 24-27 in my own index). These are public training labs, not client work and not real targets. No per-instance values, no secrets, no screenshots: only the patterns and the methodology.

## The point

In all four labs the web interface showed a narrow, safe slice of the API: read a price, place an order, request a password reset. The server behind it accepted far more - extra methods, extra fields, extra parameters, and it published a map of all of it. The boundary the developers imagined was the front end. The attacks work because the real boundary, the API itself, enforced less than the interface suggested.

## Pattern 1: The documentation is a map of the real surface

Lesson: API documentation is written for developers, but it is served to anyone who asks. It lists every endpoint and every method, including the ones the interface never links. A destructive operation does not become safe because no button points at it.

Evidence: the published documentation for the application listed a delete operation on the user endpoint. Sending that request while logged in as an ordinary user removed another user's account - the operation carried no authorization check of its own.

Fix: treat documentation as public, because it is. Every documented operation needs the same authorization as if it were linked from the home page. Obscurity is not a control.

## Pattern 2: The response shows fields the request never sends

Lesson: mass assignment starts with a leak. A GET response that echoes the internal object discloses fields the client never sets - pricing internals, discount state, role flags. If the server then auto-binds whatever JSON it receives to that same object, any disclosed field becomes writable.

Evidence: the checkout response contained a discount object with a percentage of zero, next to the products the interface does send. The order request itself only exists when the order is placed, so I built it by hand from the GET request, added the discount field set to 100 percent, and the server confirmed the order at no cost.

Fix: bind requests against an allowlist of fields the client is permitted to set, and serialize responses from a separate view that never includes internal state. The comparison that finds this bug is simple: diff the fields in the responses against the fields in the requests.

## Pattern 3: The Allow header names the hidden method

Lesson: an endpoint usually supports more methods than the interface uses. An OPTIONS request, or a 405 response, answers with an Allow header that names them. A method the UI never calls often has no authorization check of its own, because nobody thought of it as reachable.

Evidence: the price endpoint answered OPTIONS with Allow: GET, PATCH. GET is what the browser uses to display the price. PATCH with no body failed in stages, and each error message named exactly what was missing - first the content type, then the price parameter. Assembled from those errors, a PATCH setting the price to zero succeeded, and the order then went through at that price.

Fix: authorize per method, not per URL, and disable methods the application does not need. Error messages are part of the attack surface too: a generic error gives the attacker nothing to build with.

## Pattern 4: Your input becomes someone else's query string

Lesson: server-side parameter pollution is what happens when one server embeds user input into a request to another server without encoding it. The input stops being a value and becomes structure. An encoded ampersand decodes at the internal layer and adds a new parameter; an encoded hash truncates everything after it and reveals what the internal query contained.

Evidence: in the password reset flow, error messages were the oracle at every step. A nonexistent user and a valid one answered differently. An injected parameter produced "parameter is not supported", which proved the internal API was parsing my input as separate parameters. Truncating the query produced "field not specified", which revealed a field parameter sitting behind the username. Injecting that parameter changed what the internal API returned - it handed back the administrator's password reset token instead of the usual masked result. With the token, the administrator password was reset and the target user deleted.

Fix: encode user input before it is embedded into any server-side request, and allowlist which fields an internal API may return to a caller. An internal API is still a trust boundary; "internal" describes the network, not the input.

## What I check first now

1. Find the documentation before testing anything else: an API explorer or an OpenAPI document lists the surface the interface hides.
2. For every endpoint, ask which methods exist beyond the ones the UI calls. OPTIONS and the Allow header on a 405 answer this for free.
3. Diff GET responses against POST bodies. A field that appears in responses but is never sent is a mass assignment candidate.
4. Build requests the interface never sends by hand - a method swap on a known-good request - instead of waiting for the browser to produce them.
5. Read error messages as an oracle. Distinct errors map how the server parses input, and each one dictates the next probe.
6. Where the server forwards input into its own requests, test encoded separators (%26, %23) for pollution of the internal query string.

## Closing

None of these attacks broke the interface. The interface behaved exactly as designed in every lab; the API behind it simply trusted more than the interface asked for. That is the habit these labs build: test the API the server actually exposes - its documented surface, its unused methods, its hidden fields, its internal requests - not the slice of it the front end happens to show.
