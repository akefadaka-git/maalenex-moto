# Android Native Bridge — A2 Reconciliation Addendum

- **Parent specification:** `docs/architecture/ANDROID-NATIVE-BRIDGE-PLUGIN-A1.md`
- **A1 baseline:** `bd6fb8862e11f007a26842d31caa3216f3799fb0`
- **Stage:** A2 — architecture and implementation-contract reconciliation
- **Scope:** documentation only; no runtime implementation
- **Status:** consolidated proposed contract; Owner approval and A2 closure remain separate gates

## 1. Purpose and precedence

This addendum resolves the implementation-facing ambiguities identified during A2. It supplements, and does not replace, the frozen A1 architecture. A1 decisions remain controlling unless this addendum explicitly narrows their implementation contract.

The v1 transport remains AndroidX `WebMessageListener`. The application remains a native Android Activity hosting a packaged WebView. `addJavascriptInterface`, Capacitor migration, reflection-based plugin discovery, arbitrary intents, unrestricted device APIs, broad storage access, background location, and financial signing remain excluded.

The rules below are normative acceptance requirements for a future A3 implementation. They do not assert that the controls already exist in the repository.

## 2. Navigation, origin and Content Security Policy

### 2.1 Trusted document boundary

The only permitted top-level application document origin is:

- Scheme: `https`
- Host: `appassets.androidplatform.net`
- Port: default HTTPS port only (443; no explicit non-default port)
- User information: prohibited
- Packaged document path: `/assets/` and descendants only

The native navigation validator must parse and normalize the URL, then validate scheme, host, effective port, absence of user information, and path as separate fields. String-prefix or host-only checks are insufficient. Reject malformed URLs, encoded path traversal, unexpected query/fragment components where they can affect document routing, external main-frame navigation, and redirects away from the trusted document boundary. A new top-level document invalidates the existing bridge session before any new session can be established.

Register the message listener before loading the trusted document. Configure its allowed-origin rule for the exact HTTPS asset origin and independently validate every callback's `sourceOrigin` (scheme, host and effective port) and `isMainFrame == true`. Do not trust the listener allowlist as a substitute for callback validation. If `WebViewFeature.WEB_MESSAGE_LISTENER` is unavailable, keep the bridge unavailable; never fall back to `addJavascriptInterface`.

Subframes receive no native capability. The application must not enable file/content access or mixed content. External links must not navigate the app WebView; any future external-link handling requires a separate, narrowly scoped user-initiated policy and must not expose bridge methods.

### 2.2 Content Security Policy

Serve a CSP with the packaged application document using an HTTP response header where feasible; otherwise use a CSP meta element in the document head, with the limitations of meta-delivered CSP understood. The baseline policy is:

```
default-src 'none';
base-uri 'none';
object-src 'none';
frame-src 'none';
child-src 'none';
form-action 'self';
script-src 'self';
style-src 'self';
img-src 'self' data:;
font-src 'self';
media-src 'self';
connect-src 'self' <explicit-api-origins>;
worker-src 'self';
manifest-src 'self'
```

The placeholder `<explicit-api-origins>` must be replaced by the actual, reviewed HTTPS API origins; it is not a literal token and must never become `*`. Remove `data:` from `img-src` if the frontend does not require it. Do not add `'unsafe-eval'` or `'unsafe-inline'` to script policy. If a build dependency requires inline styles, document the exact requirement and prefer nonces or hashes over broad allowances. No wildcard subdomains, arbitrary remote scripts, or unreviewed third-party origins are permitted.

CSP and native navigation checks serve different purposes: CSP restricts document resource execution; native checks enforce which document can own the bridge. Both are required.

## 3. Bridge handshake, session, cancellation and idempotency

### 3.1 Handshake and readiness

A bridge session is bound to one WebView instance, one top-level document generation, the exact trusted origin, and the main frame. The native side generates a cryptographically random, one-time challenge for each document generation. The page sends a `bridge.hello` request containing protocol version 1, a newly generated page-instance UUID, and the challenge. Native code validates the origin/frame again, challenge freshness and single use, protocol version, and schema before returning a `bridge.ready` response with a new native session identifier, negotiated limits, and a native-generated session nonce.

The challenge expires after 30 seconds and is consumed on the first valid handshake attempt. The native session nonce and page-instance identifier are never accepted as standalone proof of origin; origin and main-frame checks are repeated for every message. No capability request is accepted before `READY`. A failed handshake leaves the bridge unavailable and returns no capability information beyond a generic protocol error.

Any top-level navigation, WebView replacement, renderer loss, Activity destruction, or trusted-document reload invalidates the session, cancels pending operations, clears event subscriptions, and rejects late callbacks. A fresh handshake is mandatory after recovery.

### 3.2 Requests, cancellation and deadlines

Every request carries protocol version, message type, UUIDv4 request ID, capability, action, issue time, and a schema-validated payload. The native side maintains a bounded pending-request table. The A1 limits remain: 64 KiB maximum UTF-8 encoded message size, 32 total in-flight requests, and a 15-second default deadline. Individual actions may declare a shorter or longer deadline, but no action may exceed 120 seconds without a separately reviewed exception.

Cancellation uses a distinct `bridge.cancel` envelope containing the target request ID and session identifier. Cancellation is best-effort: the native handler must stop cancellable work, release resources, and return a terminal cancelled result where the operation has not already completed. If an operation is not safely cancellable after its commit point, the handler must document that fact and return its actual final state; it must not falsely report rollback. Cancellation is bound to the current session and cannot cancel another page's request.

### 3.3 Idempotency and replay resistance

Request IDs provide correlation, not durable idempotency. Every state-changing action must declare an idempotency policy. Such actions require a client-generated UUID idempotency key scoped to the authenticated application session, capability and action. Native code records the key, a canonical payload digest, and operation state/result for the period required by that action.

- Same key and same canonical payload: return the existing in-progress state or the original terminal result; do not execute twice.
- Same key with a different payload, action or capability: reject with `IDEMPOTENCY_CONFLICT`.
- Missing key for an action that requires one: reject before dispatch.
- Read-only actions may omit an idempotency key unless their action contract says otherwise.

A bounded in-memory replay cache protects the active bridge session against duplicate request IDs. State-changing operations whose effects must survive process death require an action-specific durable idempotency record; an in-memory cache alone is not sufficient. Idempotency records must not contain secrets or raw sensitive payloads.

## 4. Strict JSON, schemas and rate limits

### 4.1 Parsing and schema rules

Check UTF-8 byte length before parsing. Accept exactly one JSON object at the top level; reject arrays, scalars, malformed UTF-8, trailing non-whitespace data, duplicate object keys, non-finite numbers, invalid Unicode, and excessive nesting. The parser must not silently coerce types.

Use strict per-message and per-action schemas. Reject unknown envelope fields and unknown payload fields unless a schema explicitly marks a versioned extension point. Validate UUID syntax, timestamp range, enum values, required fields, string lengths, array sizes and numeric bounds before dispatch. Never deserialize untrusted JSON directly into privileged domain objects or use reflection to select a handler. Return stable error codes without stack traces or parser internals.

Initial parser bounds: 64 KiB per message; maximum nesting depth 16; maximum 64 members per object; maximum 256 elements per array; maximum 16 KiB per string. Tighten individual action schemas further where possible. These are upper bounds, not a reason to accept unnecessarily large payloads.

### 4.2 Rate limiting and backpressure

Apply independent limits per WebView session and per capability, before expensive parsing or dispatch where possible:

- Maximum 32 concurrent requests per session, preserving the A1 limit.
- Maximum 8 concurrent requests per capability.
- Token bucket per session: burst capacity 10; refill 30 requests per 10 seconds.
- Capability-specific limits may be stricter for sensitive or expensive actions and must be declared in the registry.
- No unbounded request queue: when capacity is exhausted, reject immediately with `RATE_LIMITED` or `BUSY` and a bounded retry hint where appropriate.
- Events have a bounded queue; drop/coalesce non-critical events under pressure and never allow event production to block the UI thread.

Rate-limit violations must not disclose sensitive capability state. Repeated abuse may suspend the session; re-handshake does not reset account-level or action-level abuse controls. Rate-limit state and any stricter action limits must be covered by deterministic tests.

## 5. Per-action permissions and authorization

A capability is a namespace, not a permission grant. Every action must have a static descriptor specifying: capability/action ID, input/output schema, authentication requirement, authorization policy, Android runtime permissions (if any), recent-authentication requirement, rate limit, timeout, idempotency behavior, data classification, and whether the action is available in the current app state.

Authorization is evaluated for every invocation, not merely at plugin registration or handshake. The decision sequence is:

1. Validate trusted origin, main frame, active bridge session and message schema.
2. Resolve the exact statically registered action; reject unknown or disabled actions.
3. Verify authenticated application session when required.
4. Verify server-side authorization for account/business entitlements or sensitive records; client/native checks are defense in depth, not a replacement for backend authorization.
5. Verify app-level policy and recent owner/user authentication where required.
6. Request the minimum Android runtime permission just in time, only in a user-visible feature flow.
7. Re-check authorization and permission state immediately before the protected operation; fail closed if revoked or changed.
8. Dispatch only the narrowly scoped handler and return a minimized result.

Android permission, application authorization and recent authentication are distinct controls and must not be conflated. A permission grant does not authorize a business action. A successful biometric prompt does not grant general application access. Denial or revocation returns a stable error and must not trigger a fallback to broader access.

No background location, broad filesystem access, financial signing, arbitrary intents, contacts access or unrestricted device APIs are part of v1. Foreground location, camera, notifications and document selection remain unavailable until their individual action contracts and permission justifications are reviewed.

## 6. Secure storage and biometric boundaries

### 6.1 `storage.secure`

The P0 storage capability is app-private encrypted storage only; it is not a general file manager or a substitute for backend authorization. Use Android Keystore-protected, non-exportable key material and authenticated encryption (AES-256-GCM). Use a fresh cryptographically random IV for every encryption operation; never reuse an IV with the same key. Authenticate relevant record metadata as associated data. Store ciphertext and versioned metadata only; never store plaintext secrets in ordinary preferences, logs, crash reports or backups.

Keys must be scoped to the app installation and, where applicable, the local account. Define key versioning, rotation, invalidation, logout, account removal, uninstall/reinstall and device migration behavior. Exclude sensitive ciphertext/key material from Android backup unless a separately reviewed recovery design explicitly requires it. If a Keystore key becomes invalidated or unavailable, fail closed and require the approved recovery/reset path; never silently fall back to plaintext or a weaker key.

The implementation must document Android API-level behavior because the app's minimum SDK is 23. Tests must cover encryption/decryption, tampering, IV uniqueness, key invalidation, process restart, logout/account separation, backup exclusion and absence of secret logging. This addendum does not authorize storing payment credentials, private signing keys, financial assets or server master secrets on the device.

### 6.2 `auth.biometric`

Use AndroidX BiometricPrompt with the strongest supported authenticator policy appropriate to each action. The action descriptor must state whether strong biometrics only are required or whether device credential fallback is allowed. Do not silently broaden the allowed authenticator set.

For cryptographic operations requiring user presence, bind the Keystore operation to the authentication result through a Keystore-backed `CryptoObject`; do not treat a Boolean success callback as a reusable authorization token. Prefer per-operation authentication for sensitive key use. Authentication validity is scoped to the specific operation and short-lived; it is never a general bridge session grant.

Handle unavailable hardware, no enrollment, lockout, cancellation, enrollment changes, key invalidation and device credential fallback explicitly. No biometric template or biometric data is accessed, stored, transmitted or logged by the application. Authentication failure must not cause a weaker authentication fallback unless the action contract explicitly allows it.

Because minSdk is 23 and authenticator behavior varies by Android release, A3 must implement and test an explicit API-level compatibility matrix rather than assume a single authenticator flag works identically on all supported devices.

## 7. Events and lifecycle

Events are server/native-to-page notifications, not unsolicited permission grants or commands. Each event has protocol version, event name, event ID, timestamp, session ID and schema-validated data. Event names are statically declared by the capability registry. The page can subscribe only to events permitted for its active session.

Subscriptions are bound to the current document generation and are removed on unsubscribe, navigation, reload, renderer loss, Activity destruction or session invalidation. Native code must not retain page callbacks or WebView references beyond their lifecycle. Events must be delivered on the appropriate WebView thread, with bounded queues and explicit coalescing/drop behavior. Sensitive event data is minimized and re-authorized at delivery time when the underlying access can change.

## 8. CI probe evidence and negative-test acceptance

### 8.1 Exact runtime probe

The existing CI smoke gate must be upgraded to preserve the actual decoded result from `evaluateJavascript`. Kotlin must decode the callback's JSON-encoded string, parse it as JSON, validate types, and emit exactly one compact line:

```text
WEBVIEW_JS_PROBE_JSON {"schemaVersion":1,"readyState":"complete","title":"MaaleNex","rootPresent":true,"rootTextLength":42}
```

The values above are illustrative; CI must not hardcode the title or text length. The payload may contain only `schemaVersion`, `readyState`, `title`, `rootPresent` and `rootTextLength`. Never include page text, HTML, tokens, user data, response bodies or arbitrary page state.

### 8.2 CI acceptance gates

Before launch, clear logcat or establish an unambiguous launch timestamp/process identity. During the smoke window, capture the current app process and require:

1. `WEBVIEW_PAGE_FINISHED` for the expected packaged URL.
2. Exactly one well-formed `WEBVIEW_JS_PROBE_JSON` record for the current process/document.
3. Real JSON parsing and strict type validation.
4. `schemaVersion == 1`.
5. `readyState == "complete"`.
6. `rootPresent == true`.
7. `rootTextLength` is an integer greater than zero.
8. The exact decoded runtime JSON line is printed in the Actions log and written unchanged to `$GITHUB_STEP_SUMMARY`.
9. The probe record and relevant bounded diagnostics are uploaded as an artifact on both success and failure, where artifact upload is available.
10. No main-frame load error, `FATAL EXCEPTION`, WebView renderer-gone/crash signal, or app process death occurs during the verification window.
11. Each assertion emits an explicit PASS/FAIL marker; missing evidence is FAIL, not inferred success.

Do not synthesize a JSON record from grep fragments. Do not accept stale logcat output. Preserve failure diagnostics without logging sensitive application content.

### 8.3 Negative-test acceptance

A3 must automate the A1 abuse cases BR-001–BR-026, with these minimum proof categories:

- Origin: wrong scheme/port, user-info, lookalike host, external origin, redirect and invalid path are rejected.
- Frame: same-origin subframe is rejected; only the trusted main frame dispatches.
- Parser: malformed JSON, duplicate keys, wrong types, unknown fields, excessive depth/size and invalid UUID are rejected without crash.
- Protocol: unsupported version, invalid handshake, replay, duplicate request ID, idempotency conflict, expired challenge, cancellation and late callback are handled deterministically.
- Resource control: in-flight cap, per-capability cap, token bucket, bounded queue and renderer-loss recovery are tested.
- Authorization: missing session, unauthorized action, denied/revoked permission, stale authentication and unapproved registry entry are rejected.
- Lifecycle/events: navigation, reload, unsubscribe, Activity destruction and renderer loss remove subscriptions and prevent stale delivery.
- Privacy: logs and artifacts contain no tokens, payload secrets, page text, document contents or biometric data.
- CI probe: actual JSON payload is retained and parsed; every required field is asserted.

Unit tests alone cannot prove origin/frame enforcement. Android instrumentation tests must exercise the real WebView listener callback and verify `sourceOrigin` and `isMainFrame`. Emulator end-to-end tests must exercise the packaged app. Any failure in origin, frame, authorization, sensitive-data handling or lifecycle isolation blocks bridge activation.

## 9. A2 disposition and change control

This addendum consolidates the A2 implementation contract. It does not change runtime code and does not claim that the controls are implemented or tested. The A1 baseline remains intact.

A2 may be marked `VERIFIED-CLOSED` only after the Owner accepts this addendum as the reconciled contract and the A2 evidence register records the remaining implementation items as explicit A3 acceptance gates. Any unresolved contradiction with the actual AndroidX/WebView API or platform behavior must be recorded and resolved before the affected A3 task begins.

A3 remains a separately authorized stage. A3 implementation, dependency changes, Android permissions, runtime bridge activation, plugin activation, merge, release and deployment are not authorized by this documentation addendum. A5 remains separately gated.

### A2 closure checklist

- [x] Exact navigation and CSP requirements consolidated.
- [x] Handshake, cancellation, idempotency and event lifecycle consolidated.
- [x] Strict JSON and rate-limit requirements consolidated.
- [x] Per-action permission and authorization policy consolidated.
- [x] Secure-storage and biometric boundaries consolidated.
- [x] Exact CI probe evidence and negative-test acceptance consolidated.
- [ ] Owner acceptance of this addendum.
- [ ] A2 closure record.
- [ ] A3 authorization.

**Outcome:** A2 RECONCILIATION CONTRACT CONSOLIDATED / OWNER ACCEPTANCE PENDING / IMPLEMENTATION NOT STARTED.
