# Cookies API Documentation

## Overview

`cookies` is a Node.js library for reading and writing HTTP(S) cookies. It supports optional signing using Keygrip and provides middleware helpers for Connect/Express-style frameworks.

- Package name: `cookies`
- Entry point: `require('cookies')`
- License: MIT

## Installation

```bash
npm install cookies
```

## Quick Start

```js
const http = require('http')
const Cookies = require('cookies')
const Keygrip = require('keygrip')

const keys = new Keygrip(['keyboard cat', 'another secret'])

const server = http.createServer((req, res) => {
  const cookies = new Cookies(req, res, { keys })

  // set plain cookie
  cookies.set('theme', 'dark', { httpOnly: false })

  // set signed cookie
  cookies.set('sid', 'abc123', { signed: true })

  // read cookies
  const theme = cookies.get('theme')
  const sid = cookies.get('sid', { signed: true })

  res.end(`theme=${theme}; sid=${sid || 'none'}`)
})

server.listen(3000)
```

## Public API Surface

- Class: `Cookies`
  - `new Cookies(request, response, [options])`
  - `cookies.get(name, [options])`
  - `cookies.set(name, [value], [options])`
  - Static middleware factory: `Cookies.express(keys)` (alias: `Cookies.connect(keys)`)  
    Returns `(req, res, next) => void`
  - Static: `Cookies.Cookie` (low-level cookie utility class)

---

## `new Cookies(request, response, [options])`

Create a cookie jar bound to a single HTTP transaction.

- `request` (http.IncomingMessage): The Node.js request object
- `response` (http.ServerResponse): The Node.js response object
- `options` (object, optional):
  - `keys` (Keygrip | string[]): Keys for signing cookies. Provide a `Keygrip` instance or an array of secret strings (a `Keygrip` will be constructed).
  - `secure` (boolean): Override how secure connections are detected when setting cookies. If set, forces the library to treat the connection as secure (`true`) or insecure (`false`) when enforcing `secure` cookies. Useful behind TLS-terminating proxies.

Notes
- Passing an array of key strings or a `Keygrip` instance directly as the 3rd argument is supported but deprecated. Prefer `{ keys }` in `options`.
- The request/response objects are stored and used lazily; cookie parsing/verification is performed only on access.

---

## `cookies.get(name, [options])`

Read a cookie value from the inbound `Cookie` header.

- `name` (string): Cookie name
- `options` (object, optional):
  - `signed` (boolean, default depends on presence of `keys`): When `true`, verifies the signature cookie `name + '.sig'` using the provided Keygrip keys.

Returns
- `string | undefined`: Value if present and (when `signed: true`) valid; otherwise `undefined`.

Signature behavior when `signed: true`
- If signature matches the first key: returns the cookie value.
- If signature matches a later key: returns the cookie value and updates the signature to the first key (outbound Set-Cookie header is added).
- If signature matches none: returns `undefined` and sends an expired `name + '.sig'` cookie to delete the invalid signature.

Example
```js
const value = cookies.get('sid', { signed: true })
```

---

## `cookies.set(name, [value], [options])`

Write a cookie to the outbound `Set-Cookie` header and return the same `Cookies` instance (chainable).

- `name` (string): Cookie name
- `value` (string, optional): When omitted, an expired cookie is sent to delete the cookie
- `options` (object, optional):
  - `maxAge` (number): Milliseconds from `Date.now()` until expiry. Also exposes alias property `maxage` on `Cookie` instances.
  - `expires` (Date): Absolute expiry date. Defaults to a session cookie (no `expires`).
  - `path` (string): Defaults to `/`.
  - `domain` (string): No default.
  - `secure` (boolean): When `true`, the cookie is marked Secure and may only be sent over HTTPS. If the current connection is not secure, an error is thrown (see below).
  - `httpOnly` (boolean): Defaults to `true`.
  - `signed` (boolean): When `true`, also sends a `name + '.sig'` cookie with a Keygrip signature of `name=value`.
  - `overwrite` (boolean): When `true`, removes earlier cookies with the same name from the outgoing header for this response.

Secure connection enforcement
- If `options.secure === true` and the connection is not secure, an Error is thrown: `Cannot send secure cookie over unencrypted connection`.
- A connection is considered secure if:
  - `request.protocol === 'https'`, or
  - `request.connection.encrypted` is truthy, or
  - The constructor option `{ secure: true }` was provided (explicit override, e.g., behind a trusted proxy terminating TLS).

Examples
```js
// delete a cookie
cookies.set('sid')

// set a signed, httpOnly cookie for 1 hour
cookies.set('sid', 'abc123', { signed: true, maxAge: 60 * 60 * 1000 })

// set a secure cookie (will throw if not on HTTPS unless constructor had { secure: true })
cookies.set('pref', 'x', { secure: true, httpOnly: false })
```

---

## Express / Connect Middleware: `Cookies.express(keys)` (alias: `Cookies.connect(keys)`)

Factory that returns middleware which attaches `req.cookies` and `res.cookies` to the request/response.

- `keys` (Keygrip | string[]): Signing keys used for `signed` cookies

Usage with Express
```js
const express = require('express')
const Cookies = require('cookies')
const Keygrip = require('keygrip')

const app = express()
const keys = new Keygrip(['a', 'b'])

app.use(Cookies.express(keys))

app.get('/set', (req, res) => {
  res.cookies
    .set('unsigned', 'foo', { httpOnly: false })
    .set('signed', 'bar', { signed: true })
  res.end()
})

app.get('/', (req, res) => {
  const unsigned = req.cookies.get('unsigned')
  const signed = req.cookies.get('signed', { signed: true })
  res.end(`unsigned=${unsigned}; signed=${signed}`)
})
```

Usage behind a TLS-terminating proxy
```js
const app = express()
app.set('trust proxy', true)
app.use(Cookies.express(new Keygrip(['a', 'b'])))
app.use((req, res, next) => {
  // if your proxy sets X-Forwarded-Proto=https, many frameworks set req.protocol='https'
  // you can also force secure evaluation by constructing with { secure: true }
  res.cookies = new (require('cookies'))(req, res, { keys: new Keygrip(['a', 'b']), secure: true })
  next()
})
```

---

## Low-level `Cookies.Cookie` class

A utility used internally to build cookie headers. Exposed for advanced use-cases.

### Constructor
`new Cookies.Cookie(name, value, [attrs])`

- Validates `name` and `value` against basic RFC 7230 field-content
- `attrs` may include: `path`, `domain`, `expires`, `maxAge`, `httpOnly`, `secure`, `overwrite`

### Defaults
- `path`: `/`
- `httpOnly`: `true`
- `secure`: `false`
- `overwrite`: `false`

### Methods
- `toString(): string` → `"name=value"`
- `toHeader(): string` → Full `Set-Cookie` header value (adds attributes, converts `maxAge` to `expires`)

Example
```js
const { Cookie } = require('cookies')
const c = new Cookie('demo', '1', { httpOnly: false, path: '/demo' })
console.log(c.toHeader()) // demo=1; path=/demo
```

---

## Error Handling

- `TypeError: argument name is invalid` — cookie name failed basic RFC 7230 validation
- `TypeError: argument value is invalid` — cookie value failed validation
- `TypeError: option path is invalid` — invalid characters in `path`
- `TypeError: option domain is invalid` — invalid characters in `domain`
- `Error: Cannot send secure cookie over unencrypted connection` — attempted to set `secure: true` on a non-secure connection. Either ensure HTTPS, rely on `req.protocol` / `req.connection.encrypted`, configure your framework's proxy settings, or construct `Cookies` with `{ secure: true }` when appropriate.

---

## Additional Examples

### Node `http` end-to-end
```js
const http = require('http')
const Cookies = require('cookies')
const Keygrip = require('keygrip')
const keys = new Keygrip(['a', 'b'])

const server = http.createServer((req, res) => {
  const cookies = new Cookies(req, res, { keys })

  if (req.url === '/set') {
    cookies
      .set('unsigned', 'foo', { httpOnly: false })
      .set('signed', 'bar', { signed: true })
      .set('tampered', 'baz')
      .set('tampered.sig', 'bogus')

    res.statusCode = 302
    res.setHeader('Location', '/')
    return res.end('set')
  }

  const unsigned = cookies.get('unsigned')
  const signed = cookies.get('signed', { signed: true })
  const tampered = cookies.get('tampered', { signed: true })

  res.end(JSON.stringify({ unsigned, signed, tampered }))
})

server.listen(3000)
```

### Restify
```js
const restify = require('restify')
const Cookies = require('cookies')
const Keygrip = require('keygrip')

const server = restify.createServer()
const keys = new Keygrip(['a', 'b'])

server.get('/set', (req, res, next) => {
  const cookies = new Cookies(req, res, { keys })
  cookies
    .set('unsigned', 'foo', { httpOnly: false })
    .set('signed', 'bar', { signed: true })
  res.send({ ok: true })
  next()
})

server.get('/get', (req, res, next) => {
  const cookies = new Cookies(req, res, { keys })
  res.send({
    unsigned: cookies.get('unsigned'),
    signed: cookies.get('signed', { signed: true })
  })
  next()
})

server.listen(3000)
```

---

## Tips

- Prefer `maxAge` for relative lifetimes; the library will compute `expires`.
- Use `overwrite: true` to replace earlier same-name cookies during the same response.
- Always set `httpOnly: true` for session/auth cookies to mitigate XSS.
- Behind proxies, ensure your framework is configured to trust the proxy so `req.protocol` reflects the original client protocol, or pass `{ secure: true }` to the constructor when you know the connection is effectively secure.