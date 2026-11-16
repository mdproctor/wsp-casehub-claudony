# RemoteCC — Two Silent Bugs and a Working Binary

**Date:** 2026-04-07
**Type:** phase-update

---

## What I was trying to achieve: the deployment gate

The last entry ended with two open items: native binary unverified,
Mac Mini deployment on hold. I wanted to close those.

## What I believed going in: auth would be the hard part

`quarkus-security-webauthn` was the unknown quantity for GraalVM.
I expected reflection errors, proxy issues, something. We ran the build.

```
Finished generating 'remotecc-1.0.0-SNAPSHOT-runner' in 1m 44s.
BUILD SUCCESS
```

61 MB binary, 0.087s startup, `security-webauthn` in the installed features list.
No reflection errors. The native image concern was a non-event.

## The config bugs that never complained

The smoke test showed health UP, `/auth/register` returning 200,
`/app/` redirecting to login. Then something in the startup logs:

```
WARN Unrecognized configuration key "quarkus.webauthn.origins" was provided; it will be ignored
WARN Unrecognized configuration key "quarkus.webauthn.rp.name" was provided; it will be ignored
WARN Unrecognized configuration key "quarkus.webauthn.rp.id" was provided; it will be ignored
```

Those three properties had been in `application.properties` since the beginning.
The WebAuthn extension had been silently ignoring them — the relying party name,
the relying party ID, the allowed origin. Running with defaults the whole time.

`rp` is the standard WebAuthn spec abbreviation. It appears in the W3C spec
and every WebAuthn library. Quarkus uses `relying-party`. Claude confirmed the
correct keys by extracting the config class from the extension jar and running
`javap -verbose`:

```bash
jar xf quarkus-security-webauthn-3.9.5.jar io/.../WebAuthnRunTimeConfig.class
javap -verbose WebAuthnRunTimeConfig.class | grep Utf8
```

The constant pool shows actual field names and their `@ConfigItem(name=...)`
annotation values — the overrides that diverge from the field name convention.
Three wrong keys became three correct ones. No magic, just bytecode.

## The encryption key that went nowhere obvious

There was a second warning — different class, same pattern:

```
WARN [WebAuthnRecorder] Encryption key was not specified for persistent
WebAuthn auth, using temporary key <random-base64>=
```

Session cookies encrypted with a random key per JVM startup. Every server
restart invalidated every browser session.

Claude tracked it down the same way — bytecode on the inner class
`WebAuthnRecorder$1`. The code reads `httpConfiguration.encryptionKey`.
That field is `HttpConfiguration.encryptionKey`, annotated with
`@ConfigItem(name="auth.session.encryption-key")`.

The property you'd naturally guess — `quarkus.http.encryption-key` —
is unrecognized. Also silently ignored. The actual property:

```properties
%dev.quarkus.http.auth.session.encryption-key=<fixed-value>
```

Production needs `QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY` as an env var.
Without it, every deployment is a mass session logout.

## Where this leaves the deployment

The native binary works. The WebAuthn config is actually applied now.
Sessions survive restarts. 116 tests still passing after all three fixes.

What remains is purely operational: how the agent API key gets provisioned
on a fresh Mac Mini. That's the last thing between here and a running system.
