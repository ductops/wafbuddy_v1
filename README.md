<img width="1920" height="3607" alt="wafbuddy_v1_dashboard" src="https://github.com/user-attachments/assets/574317ef-87ba-4292-bd4e-54f1b42775cd" />
# WAF Use Case Curl Generator & Investigator

> A local-first, single-file WAF troubleshooting workbench for answering
> the question that always seems simple until 2:00 AM:
>
> **Is it the client, DNS, the WAF, TLS, the origin firewall, the origin
> itself, or the backend?**

The **WAF Use Case Curl Generator & Investigator** turns common WAF,
reverse-proxy, origin, API, and application-security troubleshooting
scenarios into clear, copy-ready diagnostic workflows.

Instead of remembering a pile of `curl` switches or rebuilding tests
from scratch, enter the protected hostname, origin details, API path,
and optional investigation values once. Then select a use case and
generate commands for Linux/macOS, Windows `curl.exe`, or native
PowerShell.

Every workflow explains what the test proves, how to run it, what
**good** looks like, what **bad or suspicious** looks like, and what to
investigate next.

------------------------------------------------------------------------

## Why This Exists

A `502 Bad Gateway` does not automatically mean **the WAF broke it**.

A `403` does not automatically mean **the application denied it**.

A successful direct-origin test does not prove that a SaaS WAF proxy can
reach the same origin. And a healthy management-plane connection does
not necessarily prove the application data path.

The Investigator is designed around **failure-boundary isolation**:

``` text
Client
  |
  v
DNS / CNAME
  |
  v
WAF / Reverse Proxy
  |
  v
TLS / SNI
  |
  v
Origin ACL / Firewall
  |
  v
Origin Web Server
  |
  v
Application / Backend / API
```

The goal is not merely to generate commands. The goal is to produce
**evidence**.

------------------------------------------------------------------------

# The Goods

## Sticky Inputs & Control Center

The main investigation inputs remain accessible at the top of the
application while you work.

Configure once, then move between investigations without repeatedly
scrolling back to find the controls.

Current controls include:

-   Protected hostname
-   Origin hostname or IP address
-   Origin port
-   WAF SaaS CNAME target
-   API path
-   JWT / Bearer token
-   WAF SaaS proxy source IPs / CIDRs
-   HTTP or HTTPS scheme
-   Shell / operating-system selection
-   Regenerate
-   Refresh current investigation
-   Reset
-   Collapse / Expand

The control center can be collapsed when screen space matters and
remembers its state locally.

## Multi-OS Command Generation

The application supports the most common troubleshooting environments:

-   **Linux / macOS** --- standard `curl`, DNS utilities, and common
    shell capabilities.
-   **Windows curl.exe** --- commands compatible with the modern
    Windows-provided curl workflow.
-   **Windows PowerShell** --- native tools such as `Invoke-WebRequest`,
    `Resolve-DnsName`, and `Test-NetConnection`.

When `curl.exe` provides a materially better diagnostic
capability---particularly TLS SNI and `--resolve` testing---the
application can recommend it even from a PowerShell workflow.

------------------------------------------------------------------------

# Investigation Library

## Public WAF URL Health

Establish a baseline for the protected application. The probe captures
HTTP status, response headers, resolved remote address, connection time,
TLS handshake time, time to first byte, and total request time.

This answers the first question: **Can the client successfully reach the
protected WAF-facing application?**

## Confirm Backend Origin Reachable

Tests the application directly at the origin while preserving the real
application hostname.

``` bash
curl --resolve "www.example.com:443:203.0.113.20" \
  "https://www.example.com/"
```

This is significantly better than requesting `https://203.0.113.20/`
because `--resolve` preserves the expected HTTP Host value and TLS SNI
hostname while forcing the connection to the selected origin IP.

It helps identify origin availability, TCP failures, TLS certificate
problems, SNI problems, origin-side 5xx responses, and application
failures independent of the WAF.

## 502 / 504 Investigator

One of the primary workflows in the application.

**WAF = 502, Origin = 200:** investigate WAF SaaS source-IP
allowlisting, upstream hostname/port, origin TLS trust, certificate
chain, SNI, WAF-to-origin DNS, mTLS expectations, and source-sensitive
reverse-proxy rules.

**WAF = 502, Origin = 502:** the backend/origin path is already
unhealthy. Investigate the origin reverse proxy, application server,
upstream service, load balancer, application logs, and backend
dependencies before changing WAF policy.

**WAF = 504, Origin = fast 200:** investigate dropped WAF proxy traffic,
firewall rules, ACLs, asymmetric routing, incorrect upstream
destination/port, and path-specific timeouts.

**TLS fails normally but succeeds with `-k`:** investigate certificate
trust, incomplete chains, expiration, SAN mismatch, SNI, and TLS
interception. `-k` is a diagnostic comparison---not the fix.

## Confirm Origin Accessible Only by WAF Proxy

This workflow validates both sides of an origin-lockdown control:

1.  The application works **through the WAF**.
2.  Direct origin access from an ordinary non-WAF source is **denied**.

The Investigator provides a place to enter the current WAF SaaS source
IP addresses or CIDRs. These are deliberately not hard-coded because
SaaS infrastructure and tenant deployment information can change.

Firewall, load-balancer, and origin logs remain important evidence for
proving that successful inbound sessions originate from the expected WAF
proxy addresses.

------------------------------------------------------------------------

# Check Point / Fog Connectivity

The application includes a dedicated Check Point management-plane
investigation.

This distinction matters:

``` text
Agent / Gateway ---> Fog
```

is not the same connection as:

``` text
WAF SaaS Proxy ---> Customer Origin
```

The Fog workflow tests DNS resolution, TCP/443 connectivity, TLS
negotiation, and HTTPS reachability.

A successful Fog test can provide evidence that the agent/gateway
management path is healthy. It does **not** independently prove that the
WAF SaaS proxy can reach the protected application's origin.

**Fog is green ≠ origin path is green.**

------------------------------------------------------------------------

# WAF SaaS CNAME / DNS Validation

The DNS investigation checks CNAME, A/AAAA records, the expected WAF
SaaS target, and subsequent HTTP behavior.

Useful for identifying stale WAF/CDN records, wrong CNAMEs, incomplete
migrations, split DNS, NXDOMAIN, SERVFAIL, resolver differences, and
propagation issues.

------------------------------------------------------------------------

# Rate-Limit Validation

The Investigator generates a controlled sequence of requests instead of
immediately throwing heavy concurrency at production.

It can expose HTTP `429`, `Retry-After`, rate-limit headers, enforcement
thresholds, increasing latency, unexpected `5xx`, and missing
enforcement.

The workflow encourages correlation with WAF telemetry to determine
whether identity is based on source IP, forwarded IP, X-Forwarded-For,
cookies, JWT claims, session identifiers, or API identity.

------------------------------------------------------------------------

# Cache Investigation

Repeated identical requests can expose caching behavior.

Evidence includes `Age`, `Cache-Control`, `ETag`, `Last-Modified`,
provider cache headers, time to first byte, and total response time.

This helps identify expected HIT/MISS transitions, unexpectedly uncached
content, dynamic or authenticated content being cached, incorrect cache
keys, query/cookie/header variation, and bypass behavior.

------------------------------------------------------------------------

# JWT Validation

The JWT workflow performs a three-way comparison:

1.  No token
2.  Valid test token
3.  Intentionally invalid token

This makes disagreements between WAF authentication, API gateway
behavior, application authorization, and token forwarding much easier to
see.

Potential investigation areas include issuer, audience, expiration,
signature validation, clock skew, Authorization-header forwarding,
header normalization, and WAF exceptions.

**Token safety:** use short-lived test credentials whenever possible.
Bearer tokens can leak through shell history, screenshots, terminal
output, investigation bundles, tickets, and chat messages.

------------------------------------------------------------------------

# Bot / CAPTCHA Validation

The application compares a browser-like request with an automation-like
request as a **bot enforcement smoke test**.

Modern bot systems use considerably more than User-Agent. They may
evaluate JavaScript execution, cookies, browser characteristics,
behavioral signals, reputation, request history, TLS/browser
fingerprints, and challenge completion.

A curl client may receive a CAPTCHA or JavaScript challenge that it
cannot solve---and that itself can be useful evidence.

------------------------------------------------------------------------

# OWASP WAF Smoke Tests

The application includes low-impact, detection-oriented payload shapes
for authorized testing, including:

-   SQL injection-shaped input
-   Reflected XSS-shaped input
-   Path traversal-shaped input
-   Command-injection-shaped input
-   Template-expression-shaped input

These tests validate WAF inspection, logging, event classification,
enforcement, and policy behavior. They are **not intended to exploit an
application**.

Prefer a dedicated harmless test endpoint. Only run active security
tests against systems you own or have explicit authorization to test.

------------------------------------------------------------------------

# API Discovery

The Investigator checks common public API descriptors and locations such
as OpenAPI, Swagger, API documentation, OpenID Connect discovery, and
GraphQL endpoints.

This is lightweight verification rather than brute-force endpoint
discovery.

Stronger API discovery evidence can come from HAR files, OpenAPI
specifications, WAF learning, API gateway telemetry, application
inventories, and browser runtime requests.

Useful findings include shadow APIs, deprecated APIs, undocumented
endpoints, accidentally public documentation, and unauthenticated
routes.

------------------------------------------------------------------------

# Third-Party API Discovery

Modern browser applications frequently communicate directly with
identity providers, payment services, analytics, maps, telemetry, SaaS
APIs, and other external services.

The Investigator performs lightweight extraction of externally
referenced hosts.

For SPAs, a browser HAR is usually stronger evidence because runtime
`fetch` and XHR requests may not appear in initial HTML.

This workflow can expose unknown data destinations, legacy APIs,
direct-to-origin calls, third-party dependencies, token exposure, and
unexpected cross-origin behavior.

------------------------------------------------------------------------

# API Threat Detection

The application includes harmless anomaly-oriented tests such as
unexpected HTTP methods, incorrect content types, and extra JSON
properties.

These help validate whether API security controls understand allowed
methods, expected schemas, parameter behavior, content types,
authentication, and anomalous requests.

Use a test endpoint that safely tolerates malformed or unexpected input.

------------------------------------------------------------------------

# Good / Bad / Next

The application deliberately does not stop at:

``` text
Here is your curl command.
Good luck.
```

Every investigation includes:

### ✓ Good

What a healthy or expected result typically resembles.

### ✕ Bad / Suspicious

Signals that deserve investigation.

### → Next

The next logical diagnostic step.

This turns a collection of commands into a repeatable troubleshooting
workflow.

------------------------------------------------------------------------

# Quick Investigation Navigation

The sticky control center includes shortcuts to WAF Health, Origin
Reachable, 502/504, CNAME, Rate Limit, Cache, JWT, Bot/CAPTCHA, OWASP,
and API Discovery.

The full investigation library remains available alongside the active
workflow.

------------------------------------------------------------------------

# Local-First Design

The Investigator is distributed as a **single HTML file**. No
application server is required.

Typical workflow:

1.  Open the HTML file in a modern browser.
2.  Enter investigation values.
3.  Select the operating system / shell.
4.  Select a use case.
5.  Copy the generated command.
6.  Run it from an authorized system.
7.  Compare the result with **Good / Bad / Next** guidance.
8.  Follow the evidence.

This makes the tool useful for labs, customer troubleshooting, screen
shares, migrations, incident triage, architecture workshops, WAF
demonstrations, and support investigations.

------------------------------------------------------------------------

# Evidence Boundary

Client-side curl and PowerShell tests can provide strong evidence about
DNS, TCP connectivity, TLS, SNI, HTTP status, headers, redirects,
timing, WAF-facing behavior, and direct-origin behavior.

They cannot independently prove every internal hop.

Some investigations still require WAF event logs, origin firewall logs,
load-balancer logs, reverse-proxy logs, application logs, packet
captures, cloud flow logs, and API gateway telemetry.

The Investigator is designed to help determine **which evidence source
to inspect next**.

------------------------------------------------------------------------

# Troubleshooting Philosophy

When something fails, resist changing five things at once.

``` text
Establish protected-path result
          |
          v
Establish direct-origin result
          |
          v
Compare the difference
          |
          v
Identify the failure boundary
          |
          v
Collect evidence there
          |
          v
Change one thing
          |
          v
Retest
```

A clean differential test is worth far more than twenty random commands.

------------------------------------------------------------------------

# Evidence Analyzer — Implemented in v1.2

The planned output-analysis workflow is now implemented.

The Investigator accepts pasted terminal output from the protected/WAF path, the direct-origin path, or both, then performs deterministic local analysis across:

```text
Client
  ↓
DNS
  ↓
WAF
  ↓
TLS / SNI
  ↓
Origin ACL
  ↓
Origin
  ↓
Backend
```

Current automatic detections include DNS resolution failure, TCP timeout/refusal/reset, TLS/certificate trust failure, HTTP status extraction, WAF-oriented `403` signals, rate-limit `429` and `Retry-After`, upstream `502`, timeout `504`, cache HIT/MISS evidence, CAPTCHA/bot challenge markers, remote IP, server header, TTFB, total timing, likely origin-side failure, and likely WAF-to-origin failure.

The strongest feature is **comparative classification**:

```text
WAF = 502 + Origin = 200
→ likely WAF → Origin boundary

WAF = 502 + Origin = 502
→ likely Origin / Backend chain

WAF = 504 + Origin = fast 200
→ likely WAF → Origin network / ACL / timeout boundary

TLS validation fails
→ likely TLS / certificate / SNI boundary
```

The analyzer renders a visual **Likely Failure Boundary** map plus confidence, parsed evidence comparison, findings, and a recommended next move.

One-click demo outputs are included so the analyzer can be exercised immediately without real customer data.

All analysis remains local in the browser. No AI service or external API is required for the deterministic analyzer.

This officially moves the project from a **command generator** into an **evidence-driven WAF troubleshooting workbench**.

------------------------------------------------------------------------


# HAR Context Correlator — Implemented in v1.3

Version 1.3 adds a third evidence source to the Investigator: **browser HAR context**.

The workflow now supports three complementary sensors:

```text
Protected/WAF CLI probe
          +
Direct-origin CLI probe
          +
Before / After HAR
          ↓
Evidence Correlation
          ↓
Client → DNS → WAF → TLS → Origin ACL → Origin → Backend
```

## Before / After HAR

Load a baseline **Before HAR**, a protected/current **After HAR**, or both. The roles can be swapped instantly without reloading the files.

A Focus selector supports:

- Compare
- Before
- After

## HAR Summary

Each HAR is summarized locally in the browser with counts for:

- total requests
- HTTP 403
- HTTP 429
- HTTP 502
- HTTP 504
- all 5xx responses
- slow requests over 1.5 seconds
- cache HIT evidence
- cache MISS/bypass evidence
- bot/CAPTCHA signals
- API-like requests
- unique hosts

## Before / After Delta Findings

The correlator highlights meaningful changes such as:

```text
502: Before 0 → After 6
504: Before 0 → After 2
403: Before 1 → After 14
429: Before 0 → After 8
Bot/CAPTCHA: Before 0 → After 3
```

These deltas strengthen the failure-boundary analysis without pretending that HAR can see private server-side hops.

## High-Signal Event Selection

The HAR view automatically surfaces requests with useful troubleshooting signals, including:

- 4xx and 5xx responses
- rate-limit signals
- bot/CAPTCHA markers
- cache HIT/MISS evidence
- slow requests

Select **Use** on an event to inspect its URL, status, timing, response headers, server header, Age, Retry-After, and related evidence.

Then choose **Use Selected Event in Analyzer** to feed that event directly into the existing evidence analyzer.

After-HAR events naturally feed the Protected/WAF lane. Before-HAR events provide baseline comparison context.

## CLI + HAR Correlation

The **Analyze CLI + HAR** action combines the existing terminal evidence analysis with HAR before/after findings.

This creates a stronger troubleshooting model:

```text
After HAR: 502 regression
Direct origin: 200
Protected curl: 502

→ Strong evidence around WAF → Origin boundary
```

Or:

```text
Before HAR: 200
After HAR: 403 + challenge markers
Protected curl: 403

→ Strong evidence around WAF policy / bot enforcement
```

## HAR Demo

A built-in **Load HAR Demo** creates a simple baseline comparison:

- Before: healthy 200 responses
- After: 502 responses through a simulated WAF-facing path

This allows the correlator to be exercised immediately without using customer data.

## HAR Evidence Boundary

HAR can provide strong evidence about:

- browser-facing request and response behavior
- HTTP status
- headers
- redirects
- timing
- caching
- rate limiting
- bot/CAPTCHA challenges
- API requests
- third-party hosts

HAR cannot independently prove:

- WAF SaaS → origin routing
- SaaS proxy source allowlisting
- private origin TLS trust
- backend application logs
- load-balancer internals
- server-side packet path

That distinction remains explicit in the UI.

---
# Security & Authorization

Use this application responsibly.

Active security payloads, malformed API requests, rate-limit testing,
bot testing, and similar probes should only be used against systems you
own or have explicit authorization to test.

Start conservatively. Do not turn a diagnostic test into a load test. Do
not expose production credentials in shell history or shared
troubleshooting artifacts.

------------------------------------------------------------------------

# Current Artifact

``` text
waf-use-case-curl-generator-investigator-v1.3-har-context-correlator.html
```

Current design goals:

**Clean · Visual · Local-first · Portable · Minimal dependencies ·
Copy/paste friendly · Windows friendly · Linux friendly · Evidence
focused · Useful during a live troubleshooting session**

------------------------------------------------------------------------

## Bottom Line

The WAF Use Case Curl Generator & Investigator exists to make one
difficult question easier:

> **Where is the request actually breaking?**

**Generate the probe. Capture the evidence. Compare the paths. Find the
boundary. Then fix the right thing.**
