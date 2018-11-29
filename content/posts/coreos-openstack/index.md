---
title: Run CoreOS on OpenStack
author: chris
date: 2015-04-28
template: article.jade
---

This guide will walk you through downloading CoreOS for OpenStack, importing in OpenStack  with `glance` and start your first CoreOS cluster with the `nova` tool.

## Upload the Image

Personally, I use the [OpenStack Docker CLI image](https://github.com/chris-rock/openstack-cli), that provides the nova and glance tool and is [described here](http://lollyrock.com/articles/openstack-cli-docker/). Once you are able to connect to OpenStack, you need to download the CoreOS image and bunzip it.

```bash
# download stable channel
$ wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
# extract image
$ bunzip2 coreos_production_openstack_image.img.bz2
```

Now, upload the CoreOS image via glance:

```bash
$ glance image-create --name CoreOS \
  --container-format bare \
  --disk-format qcow2 \
  --progress \
  --file coreos_production_openstack_image.img \
  --is-public True
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ed8eef431a77f11c3bea501574c590fe     |
| container_format | bare                                 |
| created_at       | 2015-04-22T13:24:42                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 5f6add97-7f21-407b-8f17-c887603efacb |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CoreOS                               |
| owner            | eef34004eb0f4f5ca722a29efd292d70     |
| protected        | False                                |
| size             | 420413440                            |
| status           | active                               |
| updated_at       | 2015-04-22T13:31:49                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
```

## Verify your image

Verify that your newly uploaded image is available in glance.

```
$ glance image-list
+--------------------------------------+---------------------------------+-------------+------------------+-------------+--------+
| ID                                   | Name                            | Disk Format | Container Format | Size        | Status |
+--------------------------------------+---------------------------------+-------------+------------------+-------------+--------+
| 5f6add97-7f21-407b-8f17-c887603efacb | CoreOS                          | qcow2       | bare             | 420413440   | active |
+--------------------------------------+---------------------------------+-------------+------------------+-------------+--------+
```

## Retrieve a new discovery token for the CoreOS cluster

This guide used the etcd instance provided by CoreOS to manage the cluster. This eases the quickstart of a CoreOS cluster. Request a new cluster token via:

```
$ curl -w "\n" 'https://discovery.etcd.io/new?size=3'
https://discovery.etcd.io/24a9bafd557e8c45f89f547de6564ae5
```

## Create a cloud config file

Create a local `cloud-config.yaml` and replace the `https://discovery.etcd.io/<token>` with your discovery url like 'https://discovery.etcd.io/24a9bafd557e8c45f89f547de6564ae5'

```yaml
#cloud-config

coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new?size=3
    # specify the intial size of your cluster with ?size=X
    discovery: https://discovery.etcd.io/<token>
    # multi-region and multi-cloud deployments need to use $public_ipv4
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
ssh_authorized_keys:
  # include one or more SSH public keys
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...
```

More detailed configuration options are available at [CoreOS Docs](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/)

## Create a new CoreOS cluster

With the cloud-init file ready, we start a 3-instance cluster via `nova`. You may need to adapt the the flavor or security group names for your setup.

```
nova boot \
--user-data ./cloud-config.yaml \
--image CoreOS \
--key-name coreos \
--flavor m1.medium \
--num-instances 3 \
--security-groups default coreos
```

After a successful run, all three instances are ready in OpenStack.

```
$ nova list
+--------------------------------------+---------------------------------------------+--------+------------+-------------+-------------------------------------+
| ID                                   | Name                                        | Status | Task State | Power State | Networks                            |
+--------------------------------------+---------------------------------------------+--------+------------+-------------+-------------------------------------+
| 2ed7d9ac-0449-4841-a780-cf9131740475 | coreos-2ed7d9ac-0449-4841-a780-cf9131740475 | ACTIVE | -          | Running     | default=192.168.0.12                |
| b919702e-be2f-4fcb-96c2-88fe9d14d320 | coreos-b919702e-be2f-4fcb-96c2-88fe9d14d320 | ACTIVE | -          | Running     | default=192.168.0.11                |
| c65cf227-05e3-46e6-9f11-a16ee6f13f4d | coreos-c65cf227-05e3-46e6-9f11-a16ee6f13f4d | ACTIVE | -          | Running     | default=192.168.0.13, 185.27.183.99 |
+--------------------------------------+---------------------------------------------+--------+------------+-------------+-------------------------------------+
```

## Verify the CoreOS cluster is registered in fleet

SSH into CoreOS:

```
ssh core@185.27.183.99
Last login: Wed Apr 22 14:03:50 2015 from 217.247.74.212
CoreOS stable (633.1.0)
core@coreos-c65cf227-05e3-46e6-9f11-a16ee6f13f4d ~ $
```

and verify that all nodes are registered properly in fleet:

```
core@coreos-c65cf227-05e3-46e6-9f11-a16ee6f13f4d ~ $ fleetctl list-machines
MACHINE     IP      METADATA
2ed7d9ac... 192.168.0.12    -
b919702e... 192.168.0.11    -
c65cf227... 192.168.0.13    -
```

Now you have a running CoreOS Cluster on OpenStack.

References:

* https://coreos.com/docs/running-coreos/platforms/openstack/
