# RemoteCC — Closing the Gaps

**Date:** 2026-04-06
**Type:** phase-update

---

## What I was trying to achieve: the three items on the production checklist

The last entry ended with a list: rate limiting, session expiry, remove dev quick-login. Those were the blockers before exposing this to the internet. We started there.

We didn't finish exactly that list. The session ended up being more interesting than a checklist.

## The thing that wasn't on the list: the dashboard was open

The first thing we found wasn't rate limiting. `application.properties` protected `/api/*` and `/ws/*`. It didn't protect `/app/*`. The entire web dashboard was accessible without authentication.

Three characters fixed it — `,/app/*` in `quarkus.http.auth.permission.protected.paths`. Then `StaticFilesTest` failed immediately: it had been hitting `/app/index.html` without auth and expecting 200. Added `@TestSecurity` at the class level and it passed. The `/app/*` gap had been there since the beginning.

## Rate limiting and the JAX-RS problem

Naively, you'd add rate limiting as a JAX-RS `ContainerRequestFilter`. That would miss the WebAuthn ceremony endpoints — `/q/webauthn/login/*` and `/q/webauthn/register/*` are managed by the Quarkus extension and never go through JAX-RS. We needed a Vert.x route handler, registered via `@Observes Router` at startup.

`AuthRateLimiter` tracks per-IP attempts as a sliding window: `ConcurrentHashMap<String, ArrayDeque<Instant>>`, synchronized per deque. Ten attempts in five minutes returns 429 with `Retry-After: 300`.

The dev cookie was a one-line fix. `ApiKeyAuthMechanism` was accepting `remotecc-dev-key` unconditionally — the cookie used by `POST /auth/dev-login`. Added `&& LaunchMode.current() == LaunchMode.DEVELOPMENT`. The endpoint itself already had that guard; the cookie check didn't.

## Then I asked Claude to check the tests

After all of that was committed and green, I asked Claude to review whether the tests were actually solid. What came back wasn't comfortable.

The `AuthRateLimiterTest` unit tests covered `isRateLimited()` in isolation. Nothing tested whether the route registration in `init()` was wired correctly — no test would catch a typo in the path pattern. The 429 status, the `Retry-After` header, the JSON body: all untested. The `LaunchMode` guard on the dev cookie had no test that sent the cookie and confirmed it was rejected. Window expiry was untestable because `Instant.now()` was hardcoded.

Claude also flagged that `updateOrStoreWebAuthnCredentials()` — the actual method the Quarkus framework calls after every login — had never been exercised in tests. Only the private test helpers went through `save()`. The "update existing credential" path was untested through the real interface.

## Fixing it — and getting bitten by Quarkus

We fixed everything. The clock became injectable: `Supplier<Instant> clock = Instant::now` as a field, with a package-private `setClockForTest()`. Tests freeze time via a single-element `Instant[]` — the lambda closes over the array reference, so mutating `time[0]` advances what the code sees. No `Thread.sleep`. No `mockStatic`.

Then the HTTP-level rate limiter test ran and broke three tests in `AuthResourceTest`. All `@QuarkusTest` classes share one application instance. `AuthRateLimiter` is `@ApplicationScoped`. After driving `/auth/register` eleven times to trigger a 429, the counter held those hits. `AuthResourceTest` ran next, hit the rate limit, and got 429s instead of 403s — with no indication the cause was in a different test class.

The fix was `@AfterEach` cleanup alongside `@BeforeEach`: `rateLimiter.resetForTest()` after each test method, so the state is clean for whatever class runs next.

## Where things actually stand

116 tests. The dashboard requires a login. The rate limiter covers the WebAuthn ceremony and registration. The dev cookie is development-only. The credential store's update path is tested through the interface the framework calls. Unauthenticated requests to `/app/index.html` get redirected.

Session expiry is still not done. The native binary hasn't been built since `quarkus-security-webauthn` was added, so Mac Mini deployment is unverified. Those are next.
