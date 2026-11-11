# JWT Attacks Academy Labs: Every Attack Is a Verification Failure

**Final draft.** PortSwigger Web Security Academy, JWT attacks topic, solved as training (labs 28-33 in my own index). These are public training labs, not client work and not real targets. No per-instance values, no secrets, no screenshots: only the patterns and the methodology.

## The point

A JWT is three base64url segments: a header, a set of claims, and a signature. The first two are just JSON anyone can read and rewrite. The token is worth exactly as much as the verification behind the third segment. Every attack in this set is the same question asked four ways: what does the server actually check before it believes the claims - the signature itself, the key, the algorithm, or nothing at all?

## Pattern 1: The signature is never checked

Lesson: if the server decodes the payload and trusts it without verifying the signature, the token is not a credential, it is a form the user fills in. The same failure appears in a cleaner costume when the server accepts alg set to none: the algorithm field is part of the token, so an attacker simply declares that no signature is required.

Evidence: changing the subject claim on an otherwise untouched token granted another user's session; and a token with the signature removed and alg set to none was accepted the same way.

Fix: signature verification is not optional, and none is not an algorithm. The server pins the accepted algorithm itself, it does not take it from the token.

## Pattern 2: A weak HMAC secret is an offline password

Lesson: HS256 signs with a shared secret. If that secret is guessable, the signature can be brute-forced offline, at full speed, with no rate limit and no logs on the target - the captured token itself is the oracle. Once the secret falls, every claim is negotiable: subject, role, expiry.

Evidence: a token's signature verified against a wordlist-found secret, after which a re-signed token with an administrator subject was accepted.

Fix: an HMAC secret needs the strength of a cryptographic key, generated randomly and stored like one. Short human-chosen strings are passwords, and passwords get cracked.

## Pattern 3: The token brings its own key

Lesson: the header can carry key material - an embedded public key in jwk, or a URL in jku pointing at a key set. A server that resolves the verification key from the token being verified has handed the pen to the person signing. The kid parameter is the same idea one level down: a key identifier the server resolves, and anything resolvable (a file path, a database lookup) is an injection surface.

Evidence: a self-signed token whose header embedded my own key pair was verified with the key it carried; and a token whose jku pointed at a key set I controlled was accepted on the strength of that pointer.

Fix: verification keys come from a server-side allowlist configured out of band. Key material inside the token is data, not authority.

## Pattern 4: Algorithm confusion - the key changes jobs

Lesson: RS256 and HS256 use the same key pair in opposite ways. With RS256 the server verifies with the public key. If the server can be told to treat the token as HS256, it verifies an HMAC using that same public key as the secret - and public keys are public, so the attacker signs anything with material the server publishes itself.

Evidence: re-signing a token as HS256 with the server's public key in place of an HMAC secret produced a token the server accepted with modified claims.

Fix: one algorithm per key, enforced server-side. The type of the key and the algorithm in the header must match what the server configured, not what the request asks for.

## What I check first now

1. Decode the token. Read the header before touching the claims: alg, kid, jku, jwk.
2. Modify a claim and resend untouched. If the server accepts it, verification is absent and nothing else matters yet.
3. Strip the signature and try alg none.
4. For HS256, test the signature against weak-secret wordlists offline before assuming the secret is strong.
5. Check whether the header can steer key selection: embedded jwk, external jku, and what kid actually resolves to on the server.
6. If the application mixes RSA and HMAC anywhere, test algorithm confusion explicitly.

## Closing

None of these attacks break cryptography. Each one walks around it, through a verification step the server skipped, weakened, or outsourced to the token itself. That is the habit these labs build: when a token appears, do not start with the claims. Start with what the server believes, and why.
