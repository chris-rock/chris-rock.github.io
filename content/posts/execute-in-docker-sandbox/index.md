---
title: Execute commands in a Docker sandbox
author: Christoph Hartmann
date: 2014-09-28
tags:
  - docker
aliases:
  - /articles/execute-in-docker-sandbox/
---

If we deal a lot with data from an untrusted source, we want to operate on data in a sandbox. For example, every file we handle may includes a virus and tries to attack our machine. 

[Docker](http://docker.io/) is an amazing tool to run arbitrary commands inside a sandbox. It is useful for testing applications or building complete application images as offered by [Docker Hub](https://registry.hub.docker.com/). 

To address the use case, it is required to build containers on demand and mix them with pre-known commands (which typically fit in a Dockerfile) and custom user defined commands.

## Problem

Currently, it is quite difficult to mix both approaches in Docker. Normally you have two possibilities:

1. Create a custom Dockerfile and build a new image
2. Start a base image like Ubuntu and use the shell with arbitrary commands

The first approach has the advantage that everything is settled during build time. Docker is optimized for this approach. In Docker this would look like:

```bash
docker build -t yourimage .
``` 

The second approach is available in Docker, but optimized for human input. Quite often you do something like

```bash
docker run -i -t ubuntu /bin/bash
``` 

Why not use the second approach via your applications, where you start executing commands in a special docker container. Docker offers a API with [stdin, stdout and stderr streams](https://docs.docker.com/reference/api/docker_remote_api_v1.9/#attach-to-a-container). As mentioned earlier, an application may converts images via GraphicsMagic. It would be more secure to run those tasks in a sandbox. By using Docker, we can easily recover to an initial state. Even if an attacker would infect the system, we would recover it to a clean system. This helps to mitigate security risks. Additionally this does not leave trash behind on your host server.

By using the Docker Streams API, we are able to:

 - ✓ run arbitrary commands
 - ✓ run commands in sandbox
 - ✓ run command automatically (via Streams API)
 - ✗ know when the command finished (streams only offer text output)
 - ✗ get command error code (is not displayed in text output)

A use case, that combines both approaches:

```bash
$ docker run -i -t ubuntu:trusty /bin/bash
root@5ead76a77765:/# apt-get update
root@5ead76a77765:/# apt-get install -y graphicsmagick wget
root@5ead76a77765:/# wget -q -O avatar.jpg https://avatars3.githubusercontent.com/u/1178413?v=2&s=460
root@5ead76a77765:/# gm convert -size 120x120 avatar.jpg -resize 120x120 +profile "*" thumbnail.jpg
```

How would you do the same in an automated fashion?


## Solution

Assume you want to convert various images with changing urls. We need to execute arbitrary commands and detect, when a command is finished. This is essential, because we may not know how long a download or conversion takes. 

To detect a process exit, we need to invent something new. Since the input and the output stream are decoupled, we do not know what output corresponds to a specific input. We require a way to correlate input and output. Instead of sending the plain command via Streams API, we generate a unique id for each request. Additionally we add a specific command that outputs the completed task id and the exit code.

```javascript
// generate process id
var id = uuid.v4();

// add exit code to command
var cmd2 = command + ';echo exit task ' + id + ' $?\n';
```

I completed this approach with [docker-exec](https://github.com/chris-rock/docker-exec). Now we are able to run the commands in docker. It also uses Javascript Promises instead of plain callbacks. This enables us to easily chain commands. The example introduced above can be written as follows:

```javascript
var ds = new DockerRunner();
ds.start().then(function (stream) {
    stream.pipe(process.stdout);
    console.log('---> run apt-get update\n');
    return ds.run('apt-get update');
}).then(function () {
    console.log('---> install gm\n');
    return ds.run('apt-get install -y graphicsmagick wget');
}).then(function () {
    console.log('---> convert image\n');
    return ds.run('gm convert -size 120x120 avatar.jpg -resize 120x120 +profile "*" thumbnail.jpg');
}).then(function () {
    console.log('---> stop container\n');
    return ds.stop();
}).then(function () {
    console.log('---> Done without error\n');
    done();
}).catch(function (err) {
    console.log('Done with error\n');
    console.log(err);
});
```

If you are using [boot2docker](https://github.com/boot2docker/boot2docker) on Mac, you need to change the initialization:

```javascript
// for mac use
var ds = new DockerRunner({
    host: 'http://127.0.0.1',
    port: 4243
});
```

## Summary

With the proposed solution, we are able to use predefined docker images and use custom commands. Now, we can automate every piece.

Happy hacking.
Chris

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)