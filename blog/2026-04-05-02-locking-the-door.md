# RemoteCC — Locking the Door

**Date:** 2026-04-05
**Type:** phase-update

---

## What I was trying to achieve: authentication before calling it production-ready

The system had 106 tests, a native binary, and an end-to-end chain from Claude CLI to real tmux sessions. But every endpoint was wide open — nothing protected `/api/*`, `/ws/*`, or the dashboard. Before exposing this to the internet, that had to change.

## What I chose and why: passkeys for browsers, API key for the Agent

I wanted passkeys — Touch ID on Mac, Face ID on iPhone. No passwords, no OAuth, no third-party service. WebAuthn is phishing-resistant by design and iCloud Keychain syncs credentials across Apple devices, so registering once covers everything I own.

Quarkus ships a first-party WebAuthn extension. You implement one interface (`WebAuthnUserProvider`) and the extension owns the hard parts: challenge generation, CBOR parsing, assertion verification, session cookies. That's the right trade-off.

The Agent is a machine client — it can't participate in a WebAuthn ceremony. Pre-shared API key, `X-Api-Key` header. The server's auth filter accepts either a valid session cookie or a valid API key, so both paths coexist.

For registration: invite-only. The owner registers first (before any credentials exist, `/auth/register` works without a token), then generates invite links for trusted users — 24h, one-time-use UUIDs.

## Implementing it: five classes, a login page, and a naming mistake caught late

I brought Claude in for the implementation. We built five new classes: `InviteService` (token map, consumed on first use), `CredentialStore` (`WebAuthnUserProvider` backed by `~/.remotecc/credentials.json` with atomic temp-file-rename saves), `ApiKeyAuthMechanism` (Quarkus `HttpAuthenticationMechanism` — reads the `X-Api-Key` header), `AuthResource` (POST invite for existing users, GET register for new ones), and `ApiKeyClientFilter` (injects the API key on every outgoing Agent request).

On top of those: login and register HTML pages using the WebAuthn browser API, a dev quick-login dialog that bypasses the ceremony with a single POST (dev mode only), and `@Authenticated` on all `/api/*` routes with `@TestSecurity` to keep the existing 106 tests green.

We also caught a naming problem. The server had been using `~/.remotecc/` both as the config directory and as the default session working directory. That's awkward — a hidden config directory as your tmux workspace. We split it: `~/.remotecc/` for config and credentials, `~/remotecc-workspace/` for sessions.

## The code review finds four issues

After the implementation landed, Claude reviewed the auth code specifically.

The first flag was timing-safe comparison. `ApiKeyAuthMechanism` was using `String.equals()` to check the incoming API key. An attacker can infer how many characters are correct from response time. The fix was `MessageDigest.isEqual()`.

Second: the invite token wasn't being consumed before serving the registration page. The flow was: validate token → serve HTML → user completes WebAuthn ceremony → token never consumed. Someone with the link could register multiple accounts. One line fixed it: `inviteService.consume(token)` before returning the HTML.

Third: file permissions on `credentials.json`. The atomic save was writing the file without setting permissions. Claude added `Files.setPosixFilePermissions(path, PosixFilePermissions.fromString("rw-------"))` after each write.

Fourth: `publicKeyAlgorithm` in the stored credential record. The field had been included because the Quarkus authenticator carries an algorithm value. But `io.vertx.ext.auth.webauthn.Authenticator` has no getter or setter for it — the field is internal and can't be round-tripped. Dead field, removed.

## What's working, what isn't

Browser login and register pages exist. The Agent authenticates with an API key. Invite tokens prevent open registration. The obvious security issues are gone.

What's not done: rate limiting on auth endpoints, explicit session expiry, and the dev quick-login must be removed before this goes public. The foundation is in place; production-hardening is not.
