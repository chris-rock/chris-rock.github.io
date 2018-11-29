---
title: OpenStack CLI in Docker
author: chris
date: 2015-04-22
template: article.jade
---

Recently, I faced the issue, that I had some python modules for OpenStack had dependency issues with other python modules. In addition I use multiple machines with the OpenStack CLI and it is always a lot of effort to synchronize the software to the latest state. I could have used virtualenv, but I had issues with this setup, too. Therefore I decided to start implementing a Docker container.

## Setup

The container handles all of the pain. It uses Ubuntu Trusty as a basis and uses their pre-packaged version of the OpenStack CLI tools. To get the cli installed, you need only [Docker](https://docs.docker.com/installation/) as precondition. On Mac I recommend using [Boot2Docker](). Now, everything is in place:

```bash
docker pull chrisrock/openstack-cli
docker run -it -v $(pwd)/config:/config chrisrock/openstack-cli /bin/bash
$ source /config/example.rc
$ nova list
```

I collect my `.rc` files in a separate config folder. By using `$(pwd)/config`, I have all my configs ready in my container. `source /config/example.rc` loads the config file. Finally all OpenStack CLI tools are at your hand.

Have fun.

References:

* [https://github.com/chris-rock/openstack-cli](https://github.com/chris-rock/openstack-cli)
* [https://registry.hub.docker.com/u/chrisrock/openstack-cli/](https://registry.hub.docker.com/u/chrisrock/openstack-cli/)

