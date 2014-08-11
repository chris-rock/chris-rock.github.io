---
title: Install OpenStack CLI on Ubuntu
author: chris
date: 2014-08-03
template: article.jade
---

To setup the Open Stack Cli on a new server, you need to install Python 2.7 and the xml libraries. Once everything is prepared, the cli can be installed with:

```
pip install OPENSTACKTOOL-novaclient
```

# Installation on Ubuntu 14.04 LTS

```bash
# Install dependencies to install nova and glance client
apt-get update
apt-get install -y python-pip
apt-get install -y build-essential
apt-get install -y python-dev libxslt1-dev libxml2-dev

# Install the Open Stack Cli
pip install python-novaclient
pip install python-glanceclient
```

Now, configure your environment variables for Open Stack. Since I use multiple tenants, I am going to create a new file for each tenant. eg. tenant1.sh:

```
# tenant1.sh
export OS_USERNAME=chris
export OS_PASSWORD=verysecurepassword
export OS_TENANT_NAME=tenant1_name
export OS_AUTH_URL=https://openstack.example.com:5000/v2.0/
export OS_AUTH_STRATEGY=keystone
```

With this approach I am able to switch tenants quickly. Just `source tenant1.sh` or execute the shell script directly. Test your setup with `nova list`.

To upload a new Ubuntu cloud image with `glance`, run the following command:

```
glance image-create --name 'Ubuntu 12.04.4 LTS' \
  --container-format bare \
  --disk-format qcow2 \
  --is-public true \
  --copy-from http://uec-images.ubuntu.com/releases/12.04.4/release/ubuntu-12.04-server-cloudimg-amd64-disk1.img

```

# Installation on MacOS X

You need to install the following packages:

 - [Python 2.7](https://www.python.org/ftp/python/2.7.8/python-2.7.8-macosx10.6.dmg)
 - [Git](http://git-scm.com/download/mac)

I experienced issues with Python 2.7 shipped with Mac OS. Therefore I recommend the installation of the official version from the Python team.

After the base packages are available, open the `Terminal` and run the following commands:

```
# Install pip
curl --silent https://bootstrap.pypa.io/get-pip.py |sudo python2.7

# Install Open Stack Tools
pip install python-novaclient
pip install python-glanceclient
```

