Cookies
=======

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][downloads-url]
[![Node.js Version][node-version-image]][node-version-url]
[![Build Status][travis-image]][travis-url]

Cookies is a [node.js](http://nodejs.org/) module for getting and setting HTTP(S) cookies. Cookies can be signed to prevent tampering, using [Keygrip](https://www.npmjs.com/package/keygrip). It can be used with the built-in node.js HTTP library, or as Connect/Express middleware.

## Requirements

* [node.js](http://nodejs.org/), tested with 0.8 and 0.10

## Install

    $ npm install cookies

## Features

* **Lazy**: Since cookie verification against multiple keys could be expensive, cookies are only verified lazily when accessed, not eagerly on each request.

* **Secure**: All cookies are `httponly` by default, and cookies sent over SSL are `secure` by default. An error will be thrown if you try to send secure cookies over an insecure socket.

* **Unobtrusive**: Signed cookies are stored the same way as unsigned cookies, instead of in an obfuscated signing format. An additional signature cookie is stored for each signed cookie, using a standard naming convention (_cookie-name_`.sig`). This allows other libraries to access the original cookies without having to know the signing mechanism.

* **Agnostic**: This library is optimized for use with [Keygrip](https://www.npmjs.com/package/keygrip), but does not require it; you can implement your own signing scheme instead if you like and use this library only to read/write cookies. Factoring the signing into a separate library encourages code reuse and allows you to use the same signing library for other areas where signing is needed, such as in URLs.

## API

For the full, detailed API reference with examples, see the new documentation at `docs/API.md`.

Copyright
---------

Copyright (c) 2014 Jed Schmidt. See LICENSE.txt for details.

Send any questions or comments [here](http://twitter.com/jedschmidt).

[npm-image]: https://img.shields.io/npm/v/cookies.svg
[npm-url]: https://npmjs.org/package/cookies
[downloads-image]: https://img.shields.io/npm/dm/cookies.svg
[downloads-url]: https://npmjs.org/package/cookies
[node-version-image]: https://img.shields.io/node/v/cookies.svg
[node-version-url]: https://nodejs.org/en/download/
[travis-image]: https://img.shields.io/travis/pillarjs/cookies/master.svg
[travis-url]: https://travis-ci.org/pillarjs/cookies
