---
title: Encrypt and decrypt content with Nodejs
author: Christoph Hartmann
date: 2014-09-02
tags:
  - nodejs
  - security
aliases:
  - /articles/nodejs-encryption/
---

<style type="text/css">
.gist {
  font-size: 12px;
}
</style>

Nodejs offers great support for cryptography. Under the hood it uses openssl and ships with a [Javascript api](http://nodejs.org/api/crypto.html). Unfortunately the api is not always as intuitive as it should be, especially when you have to deal with error codes. To make you life easier, I collected various approaches for encryption with AES 256.

Update: All examples are available on Github [node-crypto-examples](https://github.com/chris-rock/node-crypto-examples), too.

## Encryption mode

The first decision is the AES encryption mode. Currently I recommend the [CTR mode](http://nodejs.org/api/crypto.html). You may want to read [Evaluation of Some Blockcipher Modes of Operation](http://www.cs.ucdavis.edu/~rogaway/papers/modes.pdf) or [On the Security of CTR + CBC-MAC](http://csrc.nist.gov/groups/ST/toolkit/BCM/documents/ccm-ad1.pdf). The next nodejs version comes with support for [GCM](http://en.wikipedia.org/wiki/Galois/Counter_Mode) to do [authenticated encryption](http://en.wikipedia.org/wiki/Authenticated_encryption). Until then you have to use approaches like Encrypt-then-MAC and combine the encryption with the generation of [SHA hashs](../nodejs-sha512/index.md).

## Examples

### Encrypt and decrypt text

<script src="https://gist.github.com/chris-rock/993d8a22c7138d1f0d2e.js"></script>

### Encrypt and decrypt buffers

<script src="https://gist.github.com/chris-rock/6cac4e422f29c28c9d88.js"></script>

### Encrypt and decrypt streams

<script src="https://gist.github.com/chris-rock/335f92742b497256982a.js"></script>

### Use GCM for authenticated encryption

If you replace `aes-256-ctr` with `aes-256-gcm` you may think everything works as expected. Unfortunately this will result with a confusing error message: `TypeError: error:00000000:lib(0):func(0):reason(0)`. 

Authenticated encryption includes a hash of the encrypted content and helps you to identify manipulated encrypted content.

You need to set the [authentication tag](https://github.com/joyent/node/blob/857975d5e7e0d7bf38577db0478d9e5ede79922e/lib/crypto.js#L238-L245) via `decrypt.setAuthTag()`, which is currently only available if you use `crypto.createCipheriv(algorithm, key, iv)` with an [initialization vector](http://en.wikipedia.org/wiki/Initialization_vector). GCM's security is dependent on choosing a unique initialization vector for each encryption.

<script src="https://gist.github.com/chris-rock/fe87dd35d6168512a2f7.js"></script>

The new GCM mode is available in nodejs 0.11. Try it with [n](https://github.com/visionmedia/n) via

```bash
npm install -g n
sudo n 0.11.13
n use 0.11.13 crypto-gcm.js
```

Also take a look at the [nodejs tests](https://github.com/joyent/node/blob/master/test/simple/test-crypto-authenticated.js#L44-L64) for more tests with different setups.

## Conclusion

I hope the samples help you to get started with nodejs encryption.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

See also:

 * [SHA 512 Hashs with nodejs](http://lollyrock.com/articles/nodejs-sha512/)
 * [Simple file uploads with Express 4](http://lollyrock.com/articles/express4-file-upload/)
 * [Applied Content Security Policy for Nginx and Nodejs](http://lollyrock.com/articles/content-security-policy/)
 * [Ready for ES6?](http://arlimus.github.io/articles/ready.for.es6/)

