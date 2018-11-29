---
title: Open Stack VM Resizing
author: chris
date: 2014-04-02
template: article.jade
---

Once in a while you need to upgrade a VM with more CPU or storage.

## Step 1: VM preparation

A normal VM running on Openstack is well prepared for flavor resizing. Our observations just brought up the network configuration as a central point of failure. Especially RedHat-based machines fix the network config in various files. e.g. for CentOS

    # ensure the vm is properly prepared
    rm /etc/udev/rules.d/70-persistent-net.rules 
    touch /etc/udev/rules.d/70-persistent-net.rules

	# remove hardware specific settings /etc/sysconfig/network-scripts
	sed -i '/UUID/d' /etc/sysconfig/network-scripts/ifcfg-eth0
	sed -i '/HWADDR/d' /etc/sysconfig/network-scripts/ifcfg-eth0
	sed -i '/IPADDR/d' /etc/sysconfig/network-scripts/ifcfg-eth0

The vm is properly prepared by now.

## Step 2: Take a snapshot

    # create a snapshot and not down the image id
    nova image-create $INSTANCEID $IMAGENAME

## Step 3: Test the snapshot

Before we delete the old machine, we need to test the snapshot to ensure the network config is properly done and we are able to log into the new machine.

## Step 4: Gather machine details

Now, we collect all the information of the existing instance

    # show instance details
    nova show $INSTANCEID

    # display the network details (you need the id and the subnet id)
    neutron net-list

    # select the new flavor id
    nova flavor-list

## Destroy the old machine

    nova delete $INSTANCEID

## Create network port

In some cases you need to keep the same ip adress. Neutron helps to assign a specific port with the new machine.

    neutron port-create --fixed-ip subnet_id=$SUBNET_ID,ip_address=$IP $NETWORK_ID

## Recreate machine

Now we are ready to re-create the machine

    #!/bin/bash
    VMNAME=machine1
    FLAVOR=4
    IMAGE=63924226-e5ca-4ee6-a663-b2c89b0aa985
    SG=ssh,http,default
    PORT=f3999e60-9e53-4929-a9b2-ed3ab8ecf25d
    KEYNAME=key
    nova boot $VMNAME --flavor=$FLAVOR --image=$IMAGE --security-groups=$SG --nic port-id=$POST --key_name=$KEYNAME

In case you do not use the specific port, switch `port-id` with `net-id` and the network id gathered with `neutron net-list`

Happy resizing.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)