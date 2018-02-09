# TripleO Manual Setup

## Intro

Manually setup TripleO on a single baremetal by running
[Quickstart](https://github.com/openstack/tripleo-quickstart)
to create the environment and following [TripleO docs](https://docs.openstack.org/tripleo-docs/latest/)
for the manual installation.

Quickstart is *ONLY* used for creating the environment and
emulate the baremetal servers.

Disclaimers:
 * This doc includes quick steps to deploy tripleo manually
 in a single baremetal server using virtual nodes
 * This is not intended to replace official documentation
 * It should work with current master commit 7e834e1 of
 https://github.com/openstack/tripleo-quickstart

## Prep

### Environment

 - Deployer machine (where you run quickstart from): deployer@`$VIRTHOST`
 - Undercloud server: `undercloud` (libvirt guest)
 - Overcloud controller: `control_0` (libvirt guest)
 - Overcloud compute: `compute_0` (libvirt guest)

### User management

On `$VIRTHOST`:
 - Create non-root user: ```sh $ useradd deployer```
 - Create /etc/sudoers.d/deployer with: ```deployer ALL=(ALL) NOPASSWD:ALL```
 - Log in as ``depoyer``: ```sh $ su - deployer```
 - Create ssh key, add to root's authorized_keys and ensure you can:
```ssh root@127.0.0.2 'uname -a'```

### Create Environment

 - Clone quickstart:
   ```sh
   $ git clone https://github.com/openstack/tripleo-quickstart.git
   $ cd tripleo-quickstart
   ```
 - Install deps
   ```sh
   $ bash quickstart.sh --install-deps
   ```
 - Run quickstart to only create libvirt guests (TripleO won't be
   configured) on localhost (use loopback address 127.0.0.2)
   ```sh
   $ bash quickstart.sh --tags all --playbook quickstart.yml 127.0.0.2
   ```

### Check Environment

 - Check libvirt guests created for stack user:
   ```sh
   $ ssh -i ~/.quickstart/id_rsa_virt_host stack@127.0.0.2
   $ virsh list --all
   Id    Name                           State
   ----------------------------------------------------
   1     undercloud                     running
   -     compute_0                      shut off
   -     control_0                      shut off
   ```
 
:white_check_mark: Environment is ready for configuring TripleO in the
libvirt guests created.

## Undercloud Installation

Steps include:
 - Access undercloud VM
 - Set up undercloud hostaname
 - Configure repositories
 - Generate undercloud config
 - Install undercloud

### Access the undercloud
```sh 
ssh -F $HOME/.quickstart/ssh.config.ansible undercloud
```

### Set up hostname

```sh
sudo hostnamectl set-hostname undercloud.localdomain
sudo hostnamectl set-hostname --transient undercloud.localdomain
```
Add it to /etc/hosts:
```
127.0.0.1   undercloud.localdomain undercloud
```

### Set up repository

:squirrel: Note: Use latest stable branch

Access https://trunk.rdoproject.org/centos7/current/ and configure
repos for the desired release. For ``Pike``:

```sh
sudo yum install -y https://trunk.rdoproject.org/centos7/current/python2-tripleo-repos-<version>.el7.centos.noarch.rpm
sudo tripleo-repos -b pike current
sudo yum install -y python-tripleoclient
```

### Generate config

Generate ~/undercloud.conf using this tool: http://ucw-bnemec.rhcloud.com/. Use default settings.

### Install undercloud

Install the undercloud:
```sh
$ openstack undercloud install
```

You should see at the end of the installation the following message:
```
#############################################################################
Undercloud install complete.
```

## Overcloud Installation

Steps include:
 - Build Images
 - Upload images
 - Set up DNS
 - Create and start a virtual BMC for each domain
 - Introspect nodes
 - Make nodes available to deploy
 - Deploy overcloud

### Source undercloud credentials
```sh 
$ source stackrc
```

### Build Image (ignore cached images)

```sh
export DIB\_YUM\_REPO\_CONF="/etc/yum.repos.d/delorean\*"
export STABLE\_RELEASE="pike"
mv overcloud-full\* /tmp
mv ironic-python-agent\* /tmp

openstack overcloud image build --no-skip
```

### Upload images

```sh
(undercloud) [stack@undercloud ~]$ openstack overcloud image upload
```

### Create a virtual BMC for each domain

```sh

[deployer@rdo-ci-fx2-02-s5 ~]$ ssh stack@127.0.0.2

sudo yum install -y https://trunk.rdoproject.org/centos7/current/python2-tripleo-repos-0.0.1-0.20171116021457.15e17a8.el7.centos.noarch.rpm
sudo tripleo-repos -b pike current

sudo yum install -y python-virtualbmc ipmitool

[stack@rdo-ci-fx2-02-s5 ~]$ vbmc add compute_0 --port 6230 --username stack --password password --libvirt-uri qemu:///session

[stack@rdo-ci-fx2-02-s5 ~]$ vbmc add control_0 --port 6220 --username stack --password password --libvirt-uri qemu:///session

[stack@rdo-ci-fx2-02-s5 ~]$ virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     undercloud                     running
 -     compute_0                      shut off
 -     control_0                      shut off
```

### Start virtual BMCs

```sh
[stack@rdo-ci-fx2-02-s5 ~]$ vbmc start compute_0
[stack@rdo-ci-fx2-02-s5 ~]$ 2017-11-15 08:26:22,833.833 74245 INFO VirtualBMC [-] Virtual BMC for domain compute_0 started

[stack@rdo-ci-fx2-02-s5 ~]$ vbmc start control_0
[stack@rdo-ci-fx2-02-s5 ~]$ 2017-11-15 08:27:00,644.644 74255 INFO VirtualBMC [-] Virtual BMC for domain control_0 started

[stack@rdo-ci-fx2-02-s5 ~]$ ps ax | grep bmc
 74245 ?        Sl     0:00 /usr/bin/python2 /usr/bin/vbmc start compute_0
 74255 ?        Sl     0:00 /usr/bin/python2 /usr/bin/vbmc start control_0
```

### (OPTIONAL) Power undercloud `virtual baremetal servers` on

```sh
[stack@rdo-ci-fx2-02-s5 ~]$ ipmitool -I lanplus -U stack -P password -H 127.0.0.1 -p 6230 power on
Chassis Power Control: Up/On

[stack@rdo-ci-fx2-02-s5 ~]$ ipmitool -I lanplus -U stack -P password -H 127.0.0.1 -p 6220 power on
Chassis Power Control: Up/On
```

### Open ports for virtual bmc

```sh
[stack@rdo-ci-fx2-02-s5 ~]$ sudo iptables -I INPUT -p udp -m udp --dport 6220 -j ACCEPT
[stack@rdo-ci-fx2-02-s5 ~]$ sudo iptables -I INPUT -p udp -m udp --dport 6230 -j ACCEPT
[stack@rdo-ci-fx2-02-s5 ~]$ sudo iptables -I INPUT -p udp -m udp --sport 6230 -j ACCEPT
[stack@rdo-ci-fx2-02-s5 ~]$ sudo iptables -I INPUT -p udp -m udp --sport 6220 -j ACCEPT

(undercloud) [stack@undercloud ~]$  ipmitool -I lanplus -H 192.168.23.1 -p 6220 -U stack -P dog8code power status
Chassis Power is on
``` 

### List virtual baremetal servers

```sh

[deployer@rdo-ci-fx2-02-s5 ~]$ ssh -F $HOME/.quickstart/ssh.config.ansible undercloud
```
### Edit inventory file instackenv.json

```json
{
  "nodes": [
      {
      "name": "control-0",
        "pm_password": "dog8code",
        "pm_type": "pxe_ipmitool",
        "pm_user": "stack",
        "pm_port": "6220",
        "pm_addr": "192.168.23.1",
            "mac": [
        "00:18:0f:19:67:f7"
      ],
      "cpu": "2",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "capabilities": "profile:control,boot_option:local"
    }
        ,
          {
      "name": "compute-0",
        "pm_password": "dog8code",
        "pm_type": "pxe_ipmitool",
        "pm_user": "stack",
        "pm_port": "6230",
        "pm_addr": "192.168.23.1",
            "mac": [
        "00:18:0f:19:67:fb"
      ],
      "cpu": "2",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "capabilities": "profile:compute,boot_option:local"
    }
        ]
}
```

### Register nodes from inventory file instackenv.json

```sh
(undercloud) [stack@undercloud ~]$ openstack overcloud node import instackenv.json 
Started Mistral Workflow tripleo.baremetal.v1.register_or_update. Execution ID: c4c808cd-83fe-43b6-92b6-bf506b8f3b91
Waiting for messages on queue 'c02f9bbb-55c1-4e02-9239-07e5c0a0ea8b' with no timeout.


Nodes set to managed.
Successfully registered node UUID 1433c630-b4de-4a86-8a33-c15bb6b17882
Successfully registered node UUID e14c8296-d806-4f89-8cc3-6e773a5ee0e5
(undercloud) [stack@undercloud ~]$ openstack baremetal node list
+--------------------------------------+-----------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name      | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+-----------+---------------+-------------+--------------------+-------------+
| 1433c630-b4de-4a86-8a33-c15bb6b17882 | control-0 | None          | power on    | manageable         | False       |
| e14c8296-d806-4f89-8cc3-6e773a5ee0e5 | compute-0 | None          | power on    | manageable         | False       |
+--------------------------------------+-----------+---------------+-------------+--------------------+-------------+
```

###  Introspect nodes and make them available for deployment (provide) 

```sh
(undercloud) [stack@undercloud ~]$ openstack overcloud node introspect --all-manageable
Waiting for introspection to finish...
Started Mistral Workflow tripleo.baremetal.v1.introspect_manageable_nodes. Execution ID: a56d5225-031d-4031-8116-cb1fc1b21c34
Waiting for messages on queue '7347c947-4f29-4abd-99fc-2a92efec370a' with no timeout.
Introspection of node 7a69ab84-6da3-4939-bfd0-7ee8846e95e4 completed. Status:SUCCESS. Errors:None
Introspection of node cf44462e-781d-4df0-a4d7-8cf99984592d completed. Status:SUCCESS. Errors:None
Successfully introspected nodes.
Nodes introspected successfully.
Introspection completed.
```

Note: Single Step command to register, introspect and provide nodes at once

```sh
(undercloud) [stack@undercloud ~]$ openstack overcloud node import --introspect --provide instackenv.json 
```


### Set up DNS
```sh
undercloud) [stack@undercloud ~]$ openstack subnet list
+--------------------------------------+-----------------+--------------------------------------+-----------------+
| ID                                   | Name            | Network                              | Subnet          |
+--------------------------------------+-----------------+--------------------------------------+-----------------+
| e8bd74b8-2b38-4f5c-bf6b-9f089c2f69d7 | ctlplane-subnet | cd151a46-e6cc-44a7-924e-5f27a5129073 | 192.168.24.0/24 |
+--------------------------------------+-----------------+--------------------------------------+-----------------+
(undercloud) [stack@undercloud ~]$ openstack subnet set e8bd74b8-2b38-4f5c-bf6b-9f089c2f69d7 --dns-nameserver 8.8.8.8

```

### Deploy overcloud

```sh
openstack overcloud deploy --templates
```
A successful deploy should end with some output like:

```
2017-12-27 18:16:24Z [overcloud]: CREATE_COMPLETE  Stack CREATE completed successfully

 Stack overcloud CREATE_COMPLETE 

Host 192.168.24.6 not found in /home/stack/.ssh/known_hosts
Overcloud Endpoint: http://192.168.24.6:5000/v2.0
Overcloud Deployed
(undercloud) [stack@undercloud ~]$
```




# References
1. https://docs.openstack.org/tripleo-quickstart/latest/readme.html#setting-up-libvirt-guests-only 
1. https://docs.openstack.org/tripleo-docs/latest/install/installation/installation.html
1. https://docs.openstack.org/tripleo-docs/latest/install/environments/virtualbmc.html
1. https://docs.openstack.org/tripleo-docs/latest/install/basic\_deployment/basic\_deployment\_cli.html
1. https://docs.openstack.org/tripleo-quickstart/latest/accessing-undercloud.html
