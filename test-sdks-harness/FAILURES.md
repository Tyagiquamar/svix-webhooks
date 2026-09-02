# AutoConfig live-test failures (2026-09-01)

Last full rerun used padded tokens and `eventTypes` on every HTTP subscribe. The remaining failures are in generated models and the shared Kotlin HTTP client, not in the AutoConfig / AutoConfigConsumer ports.

## Scoreboard

| Lang | A v1 HTTP | B v2 HTTP | C v1 poller | D v2 poller |
|---|---|---|---|---|
| JavaScript | PASS | PASS | PASS | PASS |
| Python | PASS | PASS | PASS | PASS |
| Go | PASS | PASS | PASS | PASS |
| Rust | PASS | PASS | PASS | PASS |
| Java | PASS | PASS | PASS | FAIL (null optionals) |
| C# | PASS | PASS | PASS | FAIL (`config` required) |
| Ruby | PASS | PASS | PASS | FAIL (`config` required) |
| Kotlin | PASS | PASS | FAIL (415) | FAIL (`config` required) |
| PHP | PASS | PASS | blocked | blocked |

Java D passed in that session only after the runner filled optional `DestinationIn` fields. The harness sends the same minimal body as the other SDKs, so Java D should fail again until codegen omits JSON nulls.

## Still broken

### 1. `DestinationOut.config` required (C#, Kotlin, Ruby D)

V2 poller subscribe is HTTP 200. Body is `{ "type": "pollingEndpoint", ... }` with no `config`. The generated deserializers in C#, Kotlin, and Ruby treat `config` as required and throw.

JS, Python, Go, Rust, and Java accept the same body.

Fix in codegen (or the OpenAPI schema): default missing `config` to `{}` for `pollingEndpoint`. Do not special-case this in AutoConfigConsumer.

### 2. Kotlin 415 on v1 poller subscribe

`SvixHttpClient.executeRequest` does `Json.encodeToString(reqBody).toRequestBody()` with no media type. OkHttp then omits `Content-Type: application/json`. The v1 subscribe PUT is 415.

This is the shared Kotlin HTTP client, not AutoConfig and not codegen. HTTP subscribe (A/B) still returned 200 on this server; the v1 poller subscribe PUT is the call that 415'd.

Fix in `kotlin/lib/src/main/kotlin/SvixHttpClient.kt`.

### 3. Java `DestinationIn` serializes unset optionals as JSON null

Minimal body (`PollingEndpoint` + `eventTypes`) becomes `"batchSize": null, "maxWaitSecs": null, ...`. The server 422s (`expected u16`) on v2 subscribe, which sends `DestinationIn` as-is.

V1 C still passes because that path maps into `SinkInCommon` and never posts those nulls.

Fix in codegen: Jackson `NON_NULL` (or equivalent) on generated models. Filling `batchSize`, `maxWaitSecs`, `uid`, `metadata`, `channels`, and `status` in the caller is a workaround, not the fix.

### 4. PHP `oneOf` models are empty stubs

After a PHP codegen rerun, these files are still only the `@generated` header:

- `php/src/Models/DestinationIn.php`
- `php/src/Models/DestinationOut.php`
- `php/src/Models/AutoConfigSinkType.php`
- `php/src/Models/StreamSinkIn.php` (same pattern, unused here)

`AutoConfigConsumer` already calls `DestinationIn` / `DestinationOut::fromMixed` / `AutoConfigSinkType::create`. There is no class to run. A/B only need `EndpointIn`, which generates fine.

Do not hand-fill the stubs. Fix the PHP generator for tagged unions, regenerate, then rerun C/D.

## Not SDK bugs

These showed up in the first pass and are easy to reintroduce.

Missing `eventTypes` on `EndpointIn` 422s on this org (`filterTypes` required). The harness always sets `eventTypes: [it.autoconfig.ping]`.

Tokens from a local server often embed `surl: http://localhost:8040`. That proxy was 502; the API process is `:8071`. Setup rewrites `surl` and keeps standard padding.

An earlier fixture rewrite used `.rstrip("=")` after changing `surl`. Go, Rust, and C# rejected `len % 4 == 2` payloads. JS and Python tolerated it. Real server tokens from `STANDARD.encode` are padded. Do not strip `=` in setup.

V2 subscribe binds once. A second HTTP or poller subscribe on the same token is 409 `already_bound`. Mint new tokens (rerun setup).

501 on poll means Diom is off. Infra, not an SDK bug.

## Suggested fix order

1. Kotlin `Content-Type` (one line in the HTTP client; unblocks C).
2. Codegen: default missing `DestinationOut.config`; Java omit-nulls; PHP `oneOf` models.
3. Rerun only the failing cells: C# D, Kotlin C/D, Ruby D, Java D, PHP C/D.
