---
title: Open Stack Glance Image Create
author: chris
date: 2014-04-03
template: article.jade
---

The newest version of Ubuntu is out and we need to upload the latest version to Open Stack. Of course we try to save our bandwith. There we use `-copy-from` instead of `--file openstack_image.img`:

    glance image-create --name 'Ubuntu 14.04 LTS Beta 1' \
      --container-format bare \
      --disk-format qcow2 \
      --is-public true \
      --copy-from http://uec-images.ubuntu.com/releases/14.04/beta-1/ubuntu-14.04-beta1-server-cloudimg-amd64-disk1.img


    +------------------+--------------------------------------+
    | Property         | Value                                |
    +------------------+--------------------------------------+
    | checksum         | None                                 |
    | container_format | bare                                 |
    | created_at       | 2014-04-02T13:16:20                  |
    | deleted          | False                                |
    | deleted_at       | None                                 |
    | disk_format      | qcow2                                |
    | id               | aa5ab6de-e461-4330-136e-fb149802bdc0 |
    | is_public        | True                                 |
    | min_disk         | 0                                    |
    | min_ram          | 0                                    |
    | name             | Ubuntu 14.04 LTS Beta 1              |
    | owner            | 9ab2f1a7ca521eee9c8de5a14567d377     |
    | protected        | False                                |
    | size             | 261095936                            |
    | status           | queued                               |
    | updated_at       | 2014-04-02T13:16:20                  |
    +------------------+--------------------------------------+
