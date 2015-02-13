---
title: SHA 512 Hashs with nodejs
author: chris
date: 2014-04-06
template: article.jade
---

Quite often you need to encrypt files. Recently I updated an application from encryption to [authenticated encryption](http://en.wikipedia.org/wiki/Authenticated_encryption) and used the encrypt-then-mac approach. 

To create a hash from strings you just need a few lines in nodejs:

```javascript
    // generate a hash from string
    var crypto = require('crypto'),
        text = 'hello bob',
        key = 'mysecret key'

    // create hahs
    var hash = crypto.createHmac('sha512', key)
    hash.update(text)
    var value = hash.digest('hex')

    // print result
    console.log(value);
```

The great thing about the nodejs implementation of `Hash` is the possibility to stream data directly into the hash:

```javascript
    // generate a hash from file stream
    var crypto = require('crypto'),
        fs = require('fs'),
        key = 'mysecret key'

    // open file stream
    var fstream = fs.createReadStream('./test/hmac.js');
    var hash = crypto.createHash('sha512', key);
    hash.setEncoding('hex');

    // once the stream is done, we read the values
    fstream.on('end', function() {
        hash.end();
        // print result
        console.log(hash.read());
    });

    // pipe file to hash generator
    fstream.pipe(hash);
```

Happy Hashing.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

See also:

 * [Encrypt and decrypt content with Nodejs](http://lollyrock.com/articles/nodejs-encryption/)
 * [Simple file uploads with Express 4](http://lollyrock.com/articles/express4-file-upload/)
 * [Applied Content Security Policy for Nginx and Nodejs](http://lollyrock.com/articles/content-security-policy/)
 * [Ready for ES6?](http://arlimus.github.io/articles/ready.for.es6/)
