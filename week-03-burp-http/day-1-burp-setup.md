# Day 1 — Burp Suite Setup & HTTP Request Anatomy

**Week 3 · Foundation Phase**
**Environment:** Burp Suite Community Edition · Burp's built-in browser · Windows (Lenovo, i7-1255U / 16 GB)
**Topic:** Proxy setup, reading a raw HTTP request, Intercept vs Repeater

---

## Objective

Get Burp to capture live browser traffic, then read a single raw HTTP request end to end and understand what every part does. Goal for the day: stop seeing web pages as a black box and start seeing the requests and responses underneath them.

## Scope & ethics

All observation here was done on **my own browser traffic**. For any **active** testing (resending modified requests, injecting payloads), the target must be a lab I own or an officially scoped platform (PortSwigger Web Security Academy, DVWA, Juice Shop). The live domain used for passive inspection is redacted below as `redacted.example`. This boundary is the whole reason a company can trust a tester.

## 1. Setup

1. Installed Burp Suite Community Edition on the Windows machine.
2. Opened **Proxy → Intercept → Open Browser** to use Burp's built-in Chromium.
   - Why this matters: the built-in browser already trusts Burp's CA certificate and is pre-pointed at the proxy, so there is **no need to configure FoxyProxy or import the CA manually**. One less setup step to get wrong on day one.
3. Turned **Intercept on**, loaded a page, and captured the first raw request.

## 2. Anatomy of an HTTP request

The first captured request (a homepage load). Cookie values truncated:

```http
GET / HTTP/2
Host: redacted.example
Cookie: authjs.csrf-token=e4f521...; authjs.callback-url=http%3A%2F%2Fredacted.example
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="151", "Not=A?Brand";v="99"
Sec-Ch-Ua-Platform: "Windows"
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/151.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,...
Accept-Language: en-US,en;q=0.9
Sec-Fetch-Mode: navigate
Accept-Encoding: gzip, deflate, br
```

| Part | What it is | In this request |
|------|-----------|-----------------|
| **Method** | The action the client wants. `GET` = fetch a resource; parameters ride in the URL and the body stays empty. | `GET` |
| **Request target** | The path being requested on the server. | `/` (homepage) |
| **Host** | The domain the request is meant for (one IP can host many sites, so the server needs this). | `redacted.example` |
| **Headers** | Extra metadata the browser sends so the server knows how to handle the request. | see below |
| **Cookie** | Data the browser stores and re-sends on every request to this site, so the server can recognise the client across stateless requests. | `authjs.*` (CSRF + callback helpers — *not* a login session; I was not authenticated) |
| **Body** | Raw data sent to the server. Empty on `GET`. On `POST` its format depends on the **`Content-Type`** header — commonly `application/x-www-form-urlencoded`, `multipart/form-data` (file uploads), or `application/json`. | *(empty — GET)* |
| **Parameters** | Named values the client supplies, like variables in code. Can live in the URL query string or in the body. | none on this request |

**Headers I could explain:**
- `User-Agent` — the browser's self-description (engine, version, OS).
- `Accept` / `Accept-Language` — the formats and languages the browser will take back.
- `Sec-Ch-Ua-*` — **Client Hints**: the browser *declaring* its brand, mobile status, and platform. Important point: these are **client-supplied**, so a server should never trust them for security decisions.

> Note on the cookies: the `authjs.*` prefix means the site runs **Auth.js (NextAuth)**. `authjs.csrf-token` is CSRF protection, and `authjs.callback-url` is where to return after login. There is **no session cookie here because I was not logged in** — this becomes relevant to the experiments below.

## 3. Intercept vs Repeater

- **Intercept** — holds a request *before* it reaches the server so I can read exactly what the browser sent. Hitting **Forward** sends it on unchanged (so a plain intercepted-then-forwarded GET behaves like a normal request).
- **Repeater** (`Ctrl+R` to send a request there) — lets me edit any part of the request and fire it repeatedly, watching how the response changes. This is the core manual-testing loop and the tool I'll use most.

## 4. Experiments in Repeater

I sent the homepage request to Repeater and changed one thing at a time to see what the server actually cares about.

### Experiment 1 — Remove all cookies
**Change:** deleted the entire `Cookie` header.
**Result:** `200 OK`, page returned normally.
**Why:** `/` is a **public page** that does not require authentication, and the cookies present were only CSRF/callback helpers — not a session token (I was not logged in). So removing them changes nothing for this endpoint.
**Lesson:** `200 OK` does **not** mean "something interesting happened." It often means "this page never depended on what I changed." To see cookies actually matter, I need to hit an **authenticated** endpoint and remove the *session* cookie — then I'd expect a `302` redirect to login or a `401/403`.

### Experiment 2 — Spoof the platform header
**Change:** `Sec-Ch-Ua-Platform: "Windows"` → `"Linux"`.
**Result:** `200 OK`, response unchanged.
**Why:** Client Hints are **informational and client-controlled**. The server doesn't gate the response on them — at most it might use them for analytics or to adapt layout. A value the client fully controls can't be a security boundary.
**Lesson:** **Never trust client-supplied headers.** This is exactly the mindset behind later bugs — spoofing `X-Forwarded-For` to fake an IP, or tampering with a client-set role header to reach admin functions. Today's harmless test is the same idea at its root.

## Key takeaways

1. A web page is a stack of HTTP **requests and responses**; Burp sits in the middle and lets me read and replay them.
2. I can read a raw request fluently now: method, host, headers, cookies, body, parameters.
3. **`200 OK` is not proof of impact** — it only tells me the server accepted the request. The question is always *what changed, and did it matter?*
4. **Client-controlled input (headers, cookies, params) is never trustworthy by default** — that single principle underlies most of web hacking.

## Next — Day 2
HTTP status codes in depth (2xx/3xx/4xx/5xx) and demonstrating *why HTTPS traffic is still readable inside Burp*. Active testing moves to PortSwigger / my own lab.