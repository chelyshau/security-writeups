# SSTI Academy Lab: The Template Engine Is an Interpreter

PortSwigger Web Security Academy, server-side template injection topic, solved as training (lab 35 in my own index). This is a public training lab, not client work and not a real target. I used the official solution as reference for the final step of the chain. No per-instance values, no secrets, no screenshots: only the patterns and the methodology.

## The point

A template is code. A template engine takes a string with placeholders and evaluates expressions inside it with the full power of a programming environment: variables, attributes, method calls, and through them the object graph of the server-side application. When user input is concatenated into the template itself instead of being passed to it as data, the input stops being text and becomes part of that code. The same failure appears where a product lets users edit templates by design, in a content manager or an email editor: the feature is legitimate, and the input is still code. Most SSTI attacks follow the same path: prove the input is evaluated, identify the engine, then climb from an expression to the objects the expression can reach.

## Pattern 1: Arithmetic separates evaluation from reflection

Lesson: reflected input proves nothing - a search term echoed on the page is just text. The test that separates reflection from evaluation is a computation the page cannot contain by accident. An arithmetic probe in template syntax, such as `{{7*7}}`, returning 49 means the server evaluated the expression. Until that test runs, SSTI and ordinary reflection look identical. Inside one syntax family, the string variant separates engines: `{{7*'7'}}` returns 7777777 in Jinja2 and Tornado, but 49 in Twig, because PHP converts the string operand to a number. In Freemarker, the `${7*'7'}` equivalent errors. The probe assumes a plaintext context, where input lands in the text of the template. If input lands inside an existing expression, the first step is different - close the expression with its delimiter, then append a new one.

Evidence: the injection point was a message parameter concatenated into the page's template. The probe in ERB syntax, `<%= 7*7 %>`, came back as 49 in the rendered page: the literal text of the probe was gone, replaced by its result. That one response confirmed server-side evaluation and named the syntax family at the same time, and no other family's probe was needed.

Fix: user input must reach a template as data, through the context the developer defines, never as part of the template string. If dynamic templates are unavoidable, render them with a logic-less engine and treat the template source as code in review.

## Pattern 2: The engine's own syntax introduces the engine

Lesson: many template engines fail loudly. A malformed expression produces an error that names the engine, often the version, and sometimes the exact construct it rejected. Identification is not a preliminary step - it decides the whole exploit path, because each engine has a different object model, different reachable classes, and a different documented route from expression to code execution.

Evidence: this lab needed no separate fingerprinting phase. The probe that evaluated was written in `<%= %>` syntax, so the 49 that proved evaluation narrowed the engine to that syntax family in one step. The lab description names ERB; against an unknown target, `<%= 7*7 %>` would still leave ERB, EJS, and JSP to separate. The documented behavior of the named engine defined the rest of the work: in ERB an expression is Ruby code, which set the ceiling for what the injection could reach.

Fix: production error pages must not carry engine names, versions, or parser messages. Verbose template errors hand the attacker the fingerprinting step for free.

## Pattern 3: An expression is a door into the runtime

Lesson: SSTI is rarely about the arithmetic. Template expressions evaluate in a context populated with live objects - the request, the application configuration, framework helpers - and each object exposes attributes and methods that lead to further objects. Traversing that graph from a single expression reaches configuration secrets, file access, and in many widely used engines a path to arbitrary code execution. The injection point is one expression; the attack surface is everything the expression can touch. In some engines the graph is the long way round.

Evidence: in this lab the door opened directly onto the runtime. The same expression syntax that printed 49 accepted a call to Ruby's `system()` with an operating system command. The page rendered "true", the method's return value on success, and the target file was gone. Here the object graph was skipped entirely: the expression context itself carried full language power.

Fix: render untrusted content with engines configured for it - sandboxed modes, restricted attribute access, no method calls on application objects. Pass templates the smallest context that renders the page, because everything in the context is reachable from an injection.

## Pattern 4: An unsandboxed engine hands the expression the whole runtime

Lesson: engines differ in what an expression may touch, and that difference is the control that matters. An input-level blocklist bans spellings and loses to respelling. An engine-level sandbox intercepts attribute access inside the engine and pushes the attack toward other objects the context legitimately exposes. This lab had neither control: ERB evaluated the expression as plain Ruby, so detection and code execution were the same step. Testing a bypass for a control that is not there wastes the attempt - the first question after confirming evaluation is what, if anything, restricts the expression.

Evidence: nothing in the lab filtered the input or restricted the engine - no blocklist on the names the payload used, no sandbox intercepting attribute access. The payload used the same syntax as the arithmetic probe and went in the same way, because no layer between them existed.

Fix: treat the engine's default power as the risk. Where user content must be rendered, use sandboxed modes or a logic-less engine, pass the smallest context that renders the page, and never let input become template source in a full-power engine.

## What I check first now

1. Fuzz every reflection point with template metacharacters from each syntax family and watch for evaluation, not just reflection.
2. Confirm with arithmetic - then confirm it is server-side. Check the raw response or view-source: the same 49 computed in the browser is client-side injection, not SSTI.
3. Disambiguate with engine-specific probes (the string-multiplication test separates Jinja2 from Twig) and with deliberately broken syntax - then read the errors before writing any exploit.
4. Identify the engine and version first; the documented behavior of that engine defines the realistic impact and the chain to it.
5. Map the objects reachable from the template context before reaching for code execution - configuration and file access are often the shorter path.
6. After confirming evaluation, establish what restricts the expression - an input blocklist, an engine sandbox, or nothing at all - and match the bypass to the control that is actually there.

## Closing

The template engine only did its job in this lab. It received a template containing my input and evaluated it, with everything the application had given it. The failure was upstream, in the decision to build a template out of user input at all. That is the habit this lab builds: wherever input is rendered, ask which side of the boundary it sits on - data the template displays, or code the template runs.
