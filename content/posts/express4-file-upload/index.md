---
title: Simple file uploads with Express 4
author: Christoph Hartmann
date: 2014-11-15
tags:
  - nodejs
aliases:
  - /articles/express4-file-upload/
---

[Express](http://expressjs.com/) is a great web framework for Javascript. Quite often you have to deal with file uploads. Although this may seems like a trivial point, it has its challenges, especially if everything is asynchronous. 

## Using Busboy

For some time, [Busboy](https://github.com/mscdex/busboy) was the best solution, because it uses the Javascript eventing properly. The downside is the complex setup as the sample from the github profile demonstrates:

```javascript
var http = require('http'),
    inspect = require('util').inspect;

var Busboy = require('busboy');

http.createServer(function(req, res) {
  if (req.method === 'POST') {
    var busboy = new Busboy({ headers: req.headers });
    busboy.on('file', function(fieldname, file, filename, encoding, mimetype) {
      console.log('File [' + fieldname + ']: filename: ' + filename + ', encoding: ' + encoding + ', mimetype: ' + mimetype);
      file.on('data', function(data) {
        console.log('File [' + fieldname + '] got ' + data.length + ' bytes');
      });
      file.on('end', function() {
        console.log('File [' + fieldname + '] Finished');
      });
    });
    busboy.on('field', function(fieldname, val, fieldnameTruncated, valTruncated) {
      console.log('Field [' + fieldname + ']: value: ' + inspect(val));
    });
    busboy.on('finish', function() {
      console.log('Done parsing form!');
      res.writeHead(303, { Connection: 'close', Location: '/' });
      res.end();
    });
    req.pipe(busboy);
  } else if (req.method === 'GET') {
    res.writeHead(200, { Connection: 'close' });
    res.end('<html><head></head><body>\
               <form method="POST" enctype="multipart/form-data">\
                <input type="text" name="textfield"><br />\
                <input type="file" name="filefield"><br />\
                <input type="submit">\
              </form>\
            </body></html>');
  }
}).listen(8000, function() {
  console.log('Listening for requests');
});
```

This is not fun if you need to support this complex setup for multiple routes in express. Do not get me wrong, busboy is a great module and does the job well, but it is just not the abstraction you would like to have in your project. In normal cases you `just` need a file upload.

## Use multer

Here comes [multer](https://github.com/expressjs/multer). The final piece that fits between express and busboy. It takes all complex tasks behind the scenes and offers a configurable express middleware. Let's try it out:

Set up a `package.json`

```javascript
{
  "name": "expres-upload-test",
  "description": "simple form upload test",
  "version": "0.1.0",
  "dependencies": {
    "express": "~4.0.0",
    "multer" : "0.1.6"
  },
  "devDependencies": {
  },
  "scripts": {},
  "engines": {
    "node": "0.10.x",
    "npm": ">1.4.0"
  }
}
```

To implement the server, create a `server.js` that combines express with multer. It is 17 lines including an extra hello world route.

```javascript
var express = require('express'),
    multer  = require('multer')

var app = express()
app.use(multer({ dest: './uploads/'}))

app.get('/', function(req, res){
  res.send('hello world');
});

app.post('/', function(req, res){
    console.log(req.body) // form fields
    console.log(req.files) // form files
    res.status(204).end()
});

app.listen(3000);
```

Of course you can try this by using [httpie](https://github.com/jakubroztocil/httpie):

```bash
$ http -f POST http://localhost:3000/ file@~/Downloads/test.pdf
HTTP/1.1 200 OK
Connection: keep-alive
Date: Sat, 15 Nov 2014 11:15:21 GMT
X-Powered-By: Express
content-length: 278
content-type: application/json

{
    "file": {
        "buffer": null, 
        "encoding": "7bit", 
        "extension": "pdf", 
        "fieldname": "file", 
        "mimetype": "text/plain", 
        "name": "f459076685288eed5b4b45f80a11b908.pdf", 
        "originalname": "test.pdf", 
        "path": "uploads/f459076685288eed5b4b45f80a11b908.pdf", 
        "size": 59353, 
        "truncated": false
    }
}
```

## Upload per route

The example above works well in most use cases. If you chain multiple middleware together, you will find out, that express uses streams, but this does not fully work with middleware. Events are not piped between middleware. If you use two express middlewares in a chain and each tries to catch all stream events and calls `next()` when all the eventing is done, the second middleware will not receive the required events anymore. 

In such a case you need to optimize the middleware per route. Therefore I do not recommend to use multer as a global middleware for express. Instead I would add multer to the routes that specifically require it. 

A working example would look like:

```javascript
var express = require('express'),
    multer  = require('multer')

var app = express()

app.get('/', function(req, res){
  res.send('hello world');
});

app.post('/',[ multer({ dest: './uploads/'}), function(req, res){
    console.log(req.body) // form fields
    console.log(req.files) // form files
    res.status(204).end()
}]);

app.listen(3000);
```

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

See also:

 * [Encrypt and decrypt content with Nodejs](http://lollyrock.com/articles/nodejs-encryption/)
 * [SHA 512 Hashs with nodejs](http://lollyrock.com/articles/nodejs-sha512/)
 * [Applied Content Security Policy for Nginx and Nodejs](http://lollyrock.com/articles/content-security-policy/)
 * [Ready for ES6?](http://arlimus.github.io/articles/ready.for.es6/)
