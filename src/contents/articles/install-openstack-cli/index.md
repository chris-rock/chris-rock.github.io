---
title: Install OpenStack CLI on Ubuntu
author: chris
date: 2014-08-03
template: article.jade
---

To setup the Open Stack cli on a new server, you need to install python and the xml libraries. Once everything is setup, the cli can be installed with:

```
pip install OPENSTACKTOOL-novaclient
```

The following steps are required on Ubuntu 14.04 LTS.

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

Configure your environment variables for Open Stack. Since I use multiple tenants, I am going to create a new file for each tenant. eg. tenant1.sh:

```
# tenant1.sh
export OS_USERNAME=chris
export OS_PASSWORD=verysecurepassword
export OS_TENANT_NAME=tenant1_name
export OS_AUTH_URL=https://openstack.example.com:5000/v2.0/
export OS_AUTH_STRATEGY=keystone
```

With this approach I am able to switch tenants quickly. Just `source tenant1.sh` or execute the shell script directly. Test your setup with `nova list`.

To upload Ubuntu with `glance`, run the following command:

```
glance image-create --name 'Ubuntu 12.04.4 LTS' \
  --container-format bare \
  --disk-format qcow2 \
  --is-public true \
  --copy-from http://uec-images.ubuntu.com/releases/12.04.4/release/ubuntu-12.04-server-cloudimg-amd64-disk1.img

```