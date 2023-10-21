---
title: "Deploying RDO OpenStack in a Cohesive Manner. Part 1: Undercloud Deployment and Node Preparation"
weight: 4
type: docs
prev: posts/openstack/cloudera-cdh-via-openstack-sahara
next: posts/openstack/deploying-rdo-openstack-in-a-cohesive-manner-part-2-overcloud-deployment
sidebar:
  open: true
---

![TripleO](tripleo_owl.png)

It has always bugged me that the TripleO  (which is the preferred installer for [RDO](https://www.rdoproject.org/) and [Red Hat OpenStack Platform](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/)) [documentation](https://docs.openstack.org/tripleo-docs/latest/) is so fragmented. If you don't know TripleO, it is very difficult to follow the docs and get a successful deployment.

[TripleO Quickstart](https://docs.openstack.org/tripleo-quickstart/latest/) made that simpler, but is only meant for development purposes.

I do not mean this to take away from the effort put into the documentation. The information is all there. It is an extremely difficult task to document a software like OpenStack that is so extremely flexible. OpenStack is like Burger King. Have it your way.

In this article, I'll walk through the documentation with an order of operations that leads to a successful deployment in my lab. The key here is to see order of operations and to get an idea of which deployment artifacts are important to capture in version control.

## Prerequisites
[TripleO Bare Metal Environment Documentation](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/environments/baremetal.html)

## Networks

First let's talk through networking setup. In my lab (and consistent with Red Hat's reference architecture), I created 6 VLANs for OpenStack to utilize.

* 1100 10.100.0.0/22 External 
* 1104 10.100.4.0/24 Provisioning
* 1105 10.100.5.0/24 Internal API
* 1106 10.100.6.0/24 Tenant
* 1107 10.100.7.0/24 Storage
* 1108 10.100.8.0/24 Storage Management

You also need out of band management for your physical servers if you would like to use the automated provisioning mechanisms of TripleO.

## Hardware

In order to deploy RDO, you will need a decent amount of hardware available. OpenStack is a cloud software stack and to be useful, it requires some resources.

* A virtual or physical system for the undercloud with 4-8 cores, 16-24G of ram and 80G of disk space.
* 1 or 3 Controllers with 8-n cores, 32-64G of ram, 256-512G of disk space (ideally 2 SSDs in raid 1 mirror)
* 2+ Computes with 16+ cores, 128G+ of ram, 512G of disk space (ideally 2 SSDs in raid 1 mirror). If you plan to have ephemeral local storage, then you'll want to add extra disks to be utilized later.

## Ceph

In this lab, I will deploy Ceph to back OpenStack for object, block, and ephemeral storage. If you plan to deploy Ceph, you will need either 3 dedicated ceph-storage nodes to host the Ceph OSDs, or you can hyper-converge the OSDs into your compute roles. That is what I will do here. Full ceph documentation is out of scope for this article (other than what I am specifically deploying as an example).

I will have 3 computeHCI (Hyper-converged Infrastructure) nodes. These nodes have the same specs as computes, but have an extra disk available to be a Ceph OSD.

I'm also going to cheat a bit. My 3 controllers also have a second disk that will serve as Ceph OSDs.

## Networking for your different nodes

* Undercloud needs interfaces on provisioning and external networks. I typically do two dedicated nics for these. Undercloud also needs to be able to reach out of band management interfaces for the physical machines
* Controllers need interfaces on all networks (I typically do provisioning untagged and trunk all other VLANs)
* Computes need interfaces on provisioning, internal api, tenant, and storage networks. If provisioning isn't routable or you are doing DVR, you should add an external interface to your computes
* ComputeHCI nodes add storage management to the required compute networks
* Ceph-storage nodes need interfaces on provisioning, storage and storage management (I will not be using this role in this lab)

## Deploying the Undercloud

I created a virtual machine running CentOS 8 running on ProxMox. I gave it 4 vCPUs, 24G of ram and 80G root disk. I gave it two nics, eth0 on my external network and eth1 on my provisioning network.

For the undercloud installation, I will follow this document.

[https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_undercloud.html](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_undercloud.html)

Create the stack user. This user is where all the magic will happen for the OpenStack deployment.

```
[centos@tripleo ~]$ sudo useradd stack
[centos@tripleo ~]$ sudo passwd stack
Changing password for user stack.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[centos@tripleo ~]$ echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
stack ALL=(root) NOPASSWD:ALL
[centos@tripleo ~]$ sudo chmod 0440 /etc/sudoers.d/stack
[centos@tripleo ~]$ su - stack
Password: 
[stack@tripleo ~]$
```

The doc has you set the hostname next. Mine was already set by cloud-init and is already populated in /etc/hosts the way the doc suggests.

```
[stack@tripleo ~]$ hostname -f
tripleo.kdjlab.com
[stack@tripleo ~]$ cat /etc/hosts
...
127.0.0.1 tripleo.kdjlab.com tripleo
...
::1 tripleo.kdjlab.com tripleo
...
```

I'm using CentOS 8, so you need to pay attention in the docs when it tells you the different commands for CentOS 7, 8 and RHEL.

Next it tells us to install the RDO repos. Note that the commands given have <version> in them. You have to hit the hosted repository page to get the link for the current version. As of this writing the current version for CentOS 8 is [python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch.rpm](https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch.rpm)

```
[stack@tripleo ~]$ sudo dnf install -y https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch.rpm
...
Installed:
  python3-pip-9.0.3-16.el8.noarch                               python3-setuptools-39.2.0-5.el8.noarch           python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch          
  python36-3.6.8-2.module_el8.1.0+245+c39af44f.x86_64          

Complete!
```

Next I need to enable the OpenStack release I want. I'm going to install Train in this lab. Ussuri is available, but I want to run the version that is attached to Red Hat's current LTS release of Red Hat OpenStack Platform.

```
[stack@tripleo ~]$ sudo -E tripleo-repos -b train current ceph
Removed old repo "/etc/yum.repos.d/delorean.repo"
Installed repo delorean to /etc/yum.repos.d/delorean.repo
Installed repo delorean-train-testing to /etc/yum.repos.d/delorean-train-testing.repo
Installed repo tripleo-centos-ceph-nautilus to /etc/yum.repos.d/tripleo-centos-ceph-nautilus.repo
Installed repo tripleo-centos-highavailability to /etc/yum.repos.d/tripleo-centos-highavailability.repo
Installed repo tripleo-centos-powertools to /etc/yum.repos.d/tripleo-centos-powertools.repo
Cache was expired
20 files removed
```

Now install tripleoclient and ceph-ansible. Since I am using CentOS 8, I need to install python3-triploclient.

```
[stack@tripleo ~]$ sudo yum install -y python3-tripleoclient
... #blah blah blah
Complete!
[stack@tripleo ~]$ sudo yum install -y ceph-ansible
...
Installed:
  ceph-ansible-4.0.25-1.el8.noarch
  
Complete!
```

Now I am ready to build our undercloud.conf which will define our configuration for the undercloud installation. I have created a [github repository](https://github.com/kjw3/rdo-sbx) to hold the artifacts for this deployment. I will also create two directories that I need later.

```
[stack@tripleo ~]$ cp /usr/share/python-tripleoclient/undercloud.conf.sample ~/undercloud.conf
[stack@tripleo ~]$ git clone https://github.com/kjw3/rdo-sbx.git
Cloning into 'rdo-sbx'...
remote: Enumerating objects: 34, done.
remote: Counting objects: 100% (34/34), done.
remote: Compressing objects: 100% (25/25), done.
remote: Total 34 (delta 6), reused 31 (delta 6), pack-reused 0
Unpacking objects: 100% (34/34), done.
```

The document does not go through editing the undercloud.conf. Below are the specific changes I made for my environment (and based on past experience). Anything not specified below I left in the default setup.

```
[DEFAULT]
clean_nodes = true
inspection_extras = false
local_ip = 10.100.4.5/24
undercloud_admin_host = 10.100.4.6
undercloud_debug = false
undercloud_hostname = tripleo.kdjlab.com
undercloud_nameservers = 10.99.99.12
undercloud_ntp_servers = 10.99.99.1
undercloud_public_host = 10.100.4.7

[ctlplane-subnet]
cidr = 10.100.4.0/24
dhcp_end = 10.100.4.89
dhcp_start = 10.100.4.70
dns_nameservers = 10.99.99.12
gateway = 10.100.4.1
inspection_iprange = 10.100.4.50,10.100.4.69
```

Next I will run the container image prepare command.

```
[stack@tripleo ~]$ mkdir templates
[stack@tripleo ~]$ mkdir images
[stack@tripleo ~]$ openstack tripleo container image prepare default \
>   --local-push-destination \
>   --output-env-file ~/templates/containers-prepare-parameter.yaml
# Generated with the following on 2020-09-17T14:03:53.402296
#
#   openstack tripleo container image prepare default --local-push-destination --output-env-file /home/stack/templates/containers-prepare-parameter.yaml
#

parameter_defaults:
  ContainerImagePrepare:
  - push_destination: true
    set:
      ceph_alertmanager_image: alertmanager
      ceph_alertmanager_namespace: docker.io/prom
      ceph_alertmanager_tag: v0.16.2
      ceph_grafana_image: grafana
      ceph_grafana_namespace: docker.io/grafana
      ceph_grafana_tag: 5.2.4
      ceph_image: daemon
      ceph_namespace: docker.io/ceph
      ceph_node_exporter_image: node-exporter
      ceph_node_exporter_namespace: docker.io/prom
      ceph_node_exporter_tag: v0.17.0
      ceph_prometheus_image: prometheus
      ceph_prometheus_namespace: docker.io/prom
      ceph_prometheus_tag: v2.7.2
      ceph_tag: v4.0.12-stable-4.0-nautilus-centos-7-x86_64
      default_tag: true
      name_prefix: centos-binary-
      name_suffix: ''
      namespace: docker.io/tripleotraincentos8
      neutron_driver: ovn
      rhel_containers: false
      tag: current-tripleo
    tag_from_label: rdo_version
```

For all long running commands in this process, I recommend using either tmux or screen. I'll be using tmux as screen is not available in CentOS or RHEL 8.

```
[stack@tripleo ~]$ sudo dnf install -y tmux
...
Installed:
  tmux-2.7-1.el8.x86_64                                                               

Complete!
[stack@tripleo ~]$ tmux new -s ops
```

Now I will run the undercloud install in our tmux session.

```
[stack@tripleo ~]$ openstack undercloud install                                                                                                                                            
Using /tmp/undercloud-disk-space.yaml87zw4opaansible.cfg as config file                                                                                                                    
[WARNING]: log file at /usr/share/openstack-tripleo-validations/playbooks/ansible.log is not writeable and we cannot create it, aborting                                                   
                                                                                                                                                                                           
Success! The validation passed for all hosts:                                                                                                                                              
* undercloud 
...
Install artifact is located at /home/stack/undercloud-install-20200917151334.tar.bzip2                            
########################################################                                                                                                                                                                            Deployment successful!                                                                                                                                                                                                              ########################################################                                                                                                                                                                            Writing the stack virtual update mark file /var/lib/tripleo-heat-installer/update_mark_undercloud                 
##########################################################                                                                                                                                                                          The Undercloud has been successfully installed.                                                                   
Useful files:                                                                                                                                                                                                                       Password file is at /home/stack/undercloud-passwords.conf
The stackrc file is at ~/stackrc
                                                                                                                  Use these files to interact with OpenStack services, and
ensure they are secured.                                                                                          
##########################################################
```

If you mess something up, the undercloud install is re-runnable. However, as noted in the documentation, you should be cautious about running the undercloud install if you have an overcloud deployment. At this point that isn't a concern because I do not have an overcloud deployment yet and I had a successful run of the undercloud install command.

## Preparing for the Overcloud Deployment

I now need to do a few things to get ready for the overcloud deployment. Because I am doing hyper-converged computes, I need to create a flavor for them. We need to build the overcloud images for inspection and deployment. Then we need to register the bare metal systems and inspect them.

I'll pull parts from different docs for this section.

I could not find a hyper-converged upstream doc. So I went to Red Hat's documentation for RHOSP 16.1 to view their HCI info.

[https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/html-single/hyperconverged_infrastructure_guide/index](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/html-single/hyperconverged_infrastructure_guide/index)

The following commands create and configure the flavor I need for the HCI compute nodes on the undercloud. I adjusted vCPUs and Ram to match the other flavors.

```
(undercloud) [stack@tripleo ~]$ openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 computeHCI
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| disk                       | 40                                   |
| id                         | 2751825e-0347-40ba-8093-8cfda61b2340 |
| name                       | computeHCI                           |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 4096                                 |
| rxtx_factor                | 1.0                                  |
| swap                       |                                      |
| vcpus                      | 1                                    |
+----------------------------+--------------------------------------+

(undercloud) [stack@tripleo ~]$ openstack flavor set --property "cpu_arch"="x86_64" \
> --property "capabilities:boot_option"="local" \
> --property "resources:CUSTOM_BAREMETAL"="1" \
> --property "resources:DISK_GB"="0" \
> --property "resources:MEMORY_MB"="0" \
> --property "resources:VCPU"="0" computeHCI

(undercloud) [stack@tripleo ~]$ openstack flavor set --property "capabilities:profile"="computeHCI" computeHCI
```

Next I'll build images using the following doc:

[https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_overcloud.html#prepare-your-environment](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_overcloud.html#prepare-your-environment)

Choose the image operating system. In this lab I am using CentOS 8.

```
(undercloud) [stack@tripleo ~]$ export OS_YAML="/usr/share/openstack-tripleo-common/image-yaml/overcloud-images-centos8.yaml"
```

Next it says to install the tripleo delorean repository. I already have. Then it says to run a tripleo-repos command. I already have.

Now I need to setup some environment vars.

```
(undercloud) [stack@tripleo ~]$ export DIB_YUM_REPO_CONF="/etc/yum.repos.d/delorean* /etc/yum.repos.d/tripleo-centos-*"

(undercloud) [stack@tripleo ~]$ echo $DIB_YUM_REPO_CONF
/etc/yum.repos.d/delorean.repo /etc/yum.repos.d/delorean-train-testing.repo /etc/yum.repos.d/tripleo-centos-ceph-nautilus.repo /etc/yum.repos.d/tripleo-centos-highavailability.repo /etc/yum.repos.d/tripleo-centos-powertools.repo

(undercloud) [stack@tripleo ~]$ export STABLE_RELEASE="train"
```

Now it is time to build the images.

```
(undercloud) [stack@tripleo ~]$ cd images/ 

(undercloud) [stack@tripleo images]$ openstack overcloud image build --config-file /usr/share/openstack-tripleo-common/image-yaml/overcloud-images-python3.yaml --config-file $OS_YAML
...
2020-09-18 03:52:46.508 | Build completed successfully

(undercloud) [stack@tripleo images]$ ls
ironic-python-agent.d          ironic-python-agent.kernel  ironic-python-agent.vmlinuz  overcloud-full.initrd  overcloud-full.qcow2
ironic-python-agent.initramfs  ironic-python-agent.log     overcloud-full.d             overcloud-full.log     overcloud-full.vmlinuz
```

Next let's upload them into Glance (and in PXE boot locations) on the undercloud.

```
(undercloud) [stack@tripleo images]$ openstack overcloud image upload
Image "overcloud-full-vmlinuz" was uploaded.
+--------------------------------------+------------------------+-------------+---------+--------+
|                  ID                  |          Name          | Disk Format |   Size  | Status |
+--------------------------------------+------------------------+-------------+---------+--------+
| 85e30f1c-91b9-4daa-a0d0-ee9c43ba3249 | overcloud-full-vmlinuz |     aki     | 8920200 | active |
+--------------------------------------+------------------------+-------------+---------+--------+
Image "overcloud-full-initrd" was uploaded.
+--------------------------------------+-----------------------+-------------+----------+--------+
|                  ID                  |          Name         | Disk Format |   Size   | Status |
+--------------------------------------+-----------------------+-------------+----------+--------+
| ee6f0c1f-5e1a-4dd3-bbe9-92d5f1329009 | overcloud-full-initrd |     ari     | 59048686 | active |
+--------------------------------------+-----------------------+-------------+----------+--------+
Image "overcloud-full" was uploaded.
+--------------------------------------+----------------+-------------+------------+--------+
|                  ID                  |      Name      | Disk Format |    Size    | Status |
+--------------------------------------+----------------+-------------+------------+--------+
| 38d9db92-0eb3-40c0-8f6f-deb3f4bdd739 | overcloud-full |    qcow2    | 1051301376 | active |
+--------------------------------------+----------------+-------------+------------+--------+

(undercloud) [stack@tripleo images]$ openstack image list
+--------------------------------------+------------------------+--------+
| ID                                   | Name                   | Status |
+--------------------------------------+------------------------+--------+
| 38d9db92-0eb3-40c0-8f6f-deb3f4bdd739 | overcloud-full         | active |
| ee6f0c1f-5e1a-4dd3-bbe9-92d5f1329009 | overcloud-full-initrd  | active |
| 85e30f1c-91b9-4daa-a0d0-ee9c43ba3249 | overcloud-full-vmlinuz | active |
+--------------------------------------+------------------------+--------+
```

I am now ready to import my bare metal nodes into Ironic. For this there are several options. You can utilize auto-discovery of your systems out of band management interfaces, or you can build a json file with their information. I will use the json file since I only have 8 systems to import. You can find the details for these methods in the document below.

[https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/environments/baremetal.html](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/environments/baremetal.html)

My instackenv.json file looks like the following. Notice that I have the mac addresses for the nic that I want to PXE boot, the IPMI connection information, and I have predefined on each node what profiles they will have in the overcloud (control, compute, or computeHCI).

```
(undercloud) [stack@tripleo ~]$ cat instackenv.json 
{
    "nodes": [
        {
            "name": "sbx0",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:c7:80:56",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.30",
            "capabilities": "profile:control,boot_option:local"
        },
        {
            "name": "sbx1",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:90:dc:40",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.31",
            "capabilities": "profile:control,boot_option:local"
        },
        {
            "name": "sbx2",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:cf:89:a8",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.32",
            "capabilities": "profile:control,boot_option:local"
        },
        {
            "name": "sbx3",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:90:25:4c",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.33",
            "capabilities": "profile:compute,boot_option:local"
        },
        {
            "name": "sbx4",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:90:61:88",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.34",
            "capabilities": "profile:compute,boot_option:local"
        },
        {
            "name": "sbx5",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:cf:2e:ac",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.35",
            "capabilities": "profile:computeHCI,boot_option:local"
        },
        {
            "name": "sbx6",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:90:61:4a",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.36",
            "capabilities": "profile:computeHCI,boot_option:local"
        },
        {
            "name": "sbx7",
            "pm_type": "ipmi",
            "ports": [
                {
                    "address": "0c:c4:7a:90:dd:3e",
                    "physical_network": "ctlplane"
                }
            ],
            "arch": "x86_64",
            "pm_user": "ADMIN",
            "pm_password": "ADMIN",
            "pm_addr": "10.100.0.37",
            "capabilities": "profile:computeHCI,boot_option:local"
        }
    ]
}
```

Now I am ready to import my instackenv.json file. This command registers the nodes with Ironic, enrolls them, sets them to manageable, kicks off introspection, and lastly moves them to available. Because I set clean_nodes=true in my undercloud.conf, Ironic will also run them through the cleaning process (which wipes all their disks) before making them available.

```
(undercloud) [stack@tripleo ~]$ openstack overcloud node import --introspect --provide instackenv.json
...
```

You can watch the process by running the following command. You can see that one of my nodes has already powered back off after the inspection process.

```
(undercloud) [stack@tripleo ~]$ watch -n15 openstack baremetal node list
Every 15.0s: openstack baremetal node list                                                                                                     tripleo.kdjlab.com: Fri Sep 18 12:02:03 2020

+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| 6095248c-5ef7-4653-ac74-8e3b679a688e | sbx0 | None          | power on    | manageable         | False       |
| 15a72aeb-92e6-4aa2-bbda-bcf77037d450 | sbx1 | None          | power on    | manageable         | False       |
| 38b80699-9d31-433b-ab59-0958de550267 | sbx2 | None          | power off   | manageable         | False       |
| 3bdbd0d4-4ebc-49cf-86b8-033482ddc00b | sbx3 | None          | power on    | manageable         | False       |
| 2fc7082a-2451-4dc7-901d-42edc6103a8b | sbx4 | None          | power on    | manageable         | False       |
| 6ea4d3f4-47a8-41a9-a9af-b07c8b4d9cff | sbx5 | None          | power on    | manageable         | False       |
| 99d9ba6a-417d-4ff5-87b6-4b8523a6eb0c | sbx6 | None          | power on    | manageable         | False       |
| 5c1b414f-6f32-498d-927e-30d879a5e77d | sbx7 | None          | power on    | manageable         | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
```

This is what the nodes look like in Ironic once they are ready for overcloud deployment.

```
Every 15.0s: openstack baremetal node list                                                                                                     tripleo.kdjlab.com: Fri Sep 18 12:06:02 2020

+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| 6095248c-5ef7-4653-ac74-8e3b679a688e | sbx0 | None          | power off   | available          | False       |
| 15a72aeb-92e6-4aa2-bbda-bcf77037d450 | sbx1 | None          | power off   | available          | False       |
| 38b80699-9d31-433b-ab59-0958de550267 | sbx2 | None          | power off   | available          | False       |
| 3bdbd0d4-4ebc-49cf-86b8-033482ddc00b | sbx3 | None          | power off   | available          | False       |
| 2fc7082a-2451-4dc7-901d-42edc6103a8b | sbx4 | None          | power off   | available          | False       |
| 6ea4d3f4-47a8-41a9-a9af-b07c8b4d9cff | sbx5 | None          | power off   | available          | False       |
| 99d9ba6a-417d-4ff5-87b6-4b8523a6eb0c | sbx6 | None          | power off   | available          | False       |
| 5c1b414f-6f32-498d-927e-30d879a5e77d | sbx7 | None          | power off   | available          | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
```

## Part 1 Conclusion

At this point I now have a complete undercloud with inspected and available nodes. This was the point I used to target for day 1 of my proof of concept deployments of Red Hat OpenStack Platform. There is a lot going on already. However, in the end, I will have a private cloud with a robust API layer that offers, virtualization, object, block and ephemeral storage, self-service networking, image hosting, disk management and load balancing. The best part is that the deployment of all of this will be 100% captured in a git repository that allows version control of our infrastructure.

In the next post, Deploying RDO OpenStack in a cohesive manner. Part 2: Overcloud Deployment, I will walk through template creations, my deploy script and the overcloud deployment process.