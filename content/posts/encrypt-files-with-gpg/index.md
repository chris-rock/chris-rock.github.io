---
title: Encrypt files with GPG
author: Christoph Hartmann
date: 2014-09-14
tags:
  - security
aliases:
  - /articles/encrypt-files-with-gpg
---

Although [GPG](https://www.gnupg.org/) and [GPG Tools](https://gpgtools.org/) are well known for Email encryption, the same tool-chain can be used to encrypt files. We deep dive into the command line, but everything should work with any other UI client as well.

## Password encryption with AES

```bash
# encrypt file
gpg --cipher-algo AES256 -c test.txt
# decrypt file
gpg -d test.txt.gpg
```

## Enforce message integrity check

Although default with AES, it makes sense to force the message integrity check and can be useful if you switch to other ciphers. Especially if you receive a message like 

```bash
gpg: WARNING: message was not integrity protected
```
you should enable the integrity check. The full encryption and decryption works as before:

```bash
gpg --cipher-algo AES256 --force-mdc -c test.txt
gpg -d test.txt.gpg
```

## Sign files

Since you are using the same keychain for signing and encrypting as for your emails, its very easy to sign a message. Be aware that signed messages are not encrypted.

```bash
# encrypt and sign message, you will be prompted for your password if required
$ gpg --sign test.txt
# if you have multiple accounts, you may select it with the -u parameter
$ gpg -u ABBFDCA9 --sign test.txt

You need a passphrase to unlock the secret key for
user: "Christoph Hartmann <chris@lollyrock.com>"
4096-bit RSA key, ID ABBFDCA9, created 2014-04-29

# you can verify which person encrypted the
$ gpg --verify test.txt.gpg
gpg: Signature made So 14 Sep 22:28:04 2014 CEST using RSA key ID ABBFDCA9
gpg: Good signature from "Christoph Hartmann <chris@lollyrock.com>"

```

## Encrypt files for specific persons

Now we are going to encrypt files for `john@example.com` by using his public key. 

```bash
$ gpg --cipher-algo AES256 -e --sign --recipient john@example.com test.txt

```

With this approach we are able to store files in any cloud storage. Only the recipient is able to decrypt the content. In this specific case only John is able to decrypt the content with his private key. We do not have to ship any shared password to John and everything works out of the box. By reusing the GPG infrastructure, sharing encrypted content is much simpler. 

If you are not using GPG/PGP already for your email, please do.

Cheers
Chris

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

