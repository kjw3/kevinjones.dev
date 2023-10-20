---
title: Cloudera CDH via OpenStack Sahara
type: posts
prev: posts/openshift/
sidebar:
  open: true
---

![Hadoop Header Image](hadoop-intro.png)

Recently I published the video below showing a deployment of [Cloudera CDH](https://www.cloudera.com/products/open-source/apache-hadoop/key-cdh-components.html) using [OpenStack Data Processing (Sahara)](https://wiki.openstack.org/wiki/Sahara). In the environment shown in the video, I am running Red Hat OpenStack Platform 13 and am deploying CDH v5.11.0.

{{< youtube Sf1ruAS4Rok >}}

## Prerequisites

This post does not cover deployment of OpenStack with Sahara. For those, use the following docs:

1. [Red Hat OpenStack Platform 13 Director Installation and Usage Guide](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/director_installation_and_usage/)
2. [OpenStack Data Processing Guide](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/openstack_data_processing/)

Copies of the [CDH Sahara templates](https://github.com/redhat-kejones/sahara-cdh-5.11.0) that I used for the video.

Access to a machine that has the OpenStack clients installed and an RC file or cloud config to set your authentication for the OpenStack cloud.

A prebuilt CDH 5.11.0 image. Image build documentation can be found in the OpenStack Data Processing guide listed above.

Create 3 flavors for the CDH roles. These are the three I created in my cloud.

```
$ openstack flavor list | grep cdh
+----+-------------+-------+------+-----------+-------+-----------+
| ID | Name        |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-------------+-------+------+-----------+-------+-----------+
| 10 | cdh.manager | 16384 |  200 |         0 |     8 | True      |
| 12 | cdh.master  | 40960 |  100 |         0 |     8 | True      |
| 13 | cdh.worker  | 51200 |  200 |         0 |    16 | True      |
+----+-------------+-------+------+-----------+-------+-----------+
```

## Upload and Register the CDH Image to Glance

First thing we need to do is upload the CDH image into our OpenStack Image service (Glance).

```
$ glance image-create --name cdh5110 --disk-format qcow2 --min-disk 40 --min-ram 4096 --container-format bare --visibility public --progress --file ./cdh5110.qcow2
```
Next we need to register the CDH image and tag it in Sahara. Note that cloud-user is the base user for the RHEL cloud image.

```
$ openstack dataprocessing image register --username cloud-user cdh5110
$ openstack dataprocessing image tags add cdh5110 --tags cdh 5.11.0
```

We now have a registered and tagged CDH v5.11.0 image ready for use.

```
$ openstack dataprocessing image list
+---------+--------------------------------------+------------+-------------+
| Name    | Id                                   | Username   | Tags        |
+---------+--------------------------------------+------------+-------------+
| cdh5110 | d432cf49-df1c-4904-9381-58bcdd6d1bfe | cloud-user | 5.11.0, cdh |
+---------+--------------------------------------+------------+-------------+
```

## Sahara CDH Node Group and Cluster Templates

Sahara has the concept of Node Group Templates and Cluster templates. The Node Group Templates define the different types of nodes that can be constructed in the clusters.

The Cluster template describes the mixture of node groups that will make up a single cluster.

Creating these templates is an administration task. They can be written in JSON and version controlled like I have done in the [repo mentioned in prerequisites](https://github.com/redhat-kejones/sahara-cdh-5.11.0).

For CDH, I will define four roles:

1. manager
2. master-core
3. master-additional
4. worker-nm-dn (nm = Node Manager, dn = Data Node)

You could split these up further if desired. You can use the [Sahara CDH validation rules](https://docs.openstack.org/sahara/pike/user/cdh-plugin.html) as a guide.

NOTE: Make sure to replace any references to UUIDs and Flavor IDs in the templates if you use mine for reference.

Next, create the template for Cloudera Manager role. Replace the image_id, floating_ip_pool, and flavor id with ones from your cloud.

```
$ cat manager.json
{
    "plugin_name": "cdh",
    "hadoop_version": "5.11.0",
    "node_processes": [
        "CLOUDERA_MANAGER",
        "KMS"
    ],
    "name": "cdh-5110-default-manager",
    "image_id": "d432cf49-df1c-4904-9381-58bcdd6d1bfe",
    "floating_ip_pool": "347a2783-644a-465d-9de5-adb4d972551a",
    "flavor_id": "10",
    "auto_security_group": true,
    "is_protected": false
}
$ openstack dataprocessing node group template create --json manager.json
```

Create the template for Master Core role. Replace the image_id, floating_ip_pool, and flavor id with ones from your cloud.

```
$ cat master-core.json
{
    "plugin_name": "cdh",
    "hadoop_version": "5.11.0",
    "node_processes": [
        "HDFS_NAMENODE",
        "YARN_RESOURCEMANAGER",
        "YARN_NODEMANAGER",
        "SENTRY_SERVER",
        "ZOOKEEPER_SERVER",
        "HUE_SERVER",
        "IMPALA_CATALOGSERVER"
    ],
    "name": "cdh-5110-default-master-core",
    "image_id": "d432cf49-df1c-4904-9381-58bcdd6d1bfe",
    "floating_ip_pool": "347a2783-644a-465d-9de5-adb4d972551a",
    "flavor_id": "12",
    "auto_security_group": true,
    "is_protected": false
}
$ openstack dataprocessing node group template create --json master-core.json
```

Create the template for Master Additional role. Replace the image_id, floating_ip_pool, and flavor id with ones from your cloud.

```
$ cat master-additional.json
{
    "plugin_name": "cdh",
    "hadoop_version": "5.11.0",
    "node_processes": [
        "OOZIE_SERVER",
        "YARN_JOBHISTORY",
        "YARN_NODEMANAGER",
        "HDFS_SECONDARYNAMENODE",
        "HIVE_METASTORE",
        "HIVE_SERVER2",
        "HIVE_WEBHCAT",
        "SPARK_YARN_HISTORY_SERVER",
        "SOLR_SERVER",
        "SQOOP_SERVER",
        "IMPALA_STATESTORE"
    ],
    "name": "cdh-5110-default-master-additional",
    "image_id": "d432cf49-df1c-4904-9381-58bcdd6d1bfe",
    "floating_ip_pool": "347a2783-644a-465d-9de5-adb4d972551a",
    "flavor_id": "12",
    "auto_security_group": true,
    "is_protected": false
}
$ openstack dataprocessing node group template create --json master-additional.json
```

Create the template for Worker (Node Manager, Data Node) role. Replace the image_id, floating_ip_pool, and flavor id with ones from your cloud. Also note that you may choose to change the number of volumes per node and the volume size (in GB).

```
$ cat worker-nm-dn.json
{
    "plugin_name": "cdh",
    "hadoop_version": "5.11.0",
    "node_processes": [
        "HDFS_DATANODE",
        "YARN_NODEMANAGER",
        "FLUME_AGENT",
        "IMPALAD",
        "KAFKA_BROKER"
    ],
    "name": "cdh-5110-default-nodemanager-datanode",
    "image_id": "d432cf49-df1c-4904-9381-58bcdd6d1bfe",
    "floating_ip_pool": "347a2783-644a-465d-9de5-adb4d972551a",
    "flavor_id": "13",
    "auto_security_group": true,
    "volumes_size": 200,
    "volumes_per_node": 2,
    "is_protected": false,
    "node_configs": {
      "DATANODE": {
        "dfs_datanode_du_reserved": 0
      }
    }
}
$ openstack dataprocessing node group template create --json worker-nm-dn.json
```

You should now have the four node group templates defined in Sahara. You will need their UUIDs for the cluster template.

```
$ openstack dataprocessing node group template list
+---------------------------------------+--------------------------------------+-------------+----------------+
| Name                                  | Id                                   | Plugin name | Plugin version |
+---------------------------------------+--------------------------------------+-------------+----------------+
| cdh-5110-default-master-core          | 315cd17a-4c70-4ec8-bc41-a5715edae708 | cdh         | 5.11.0         |
| cdh-5110-default-manager              | 3a82330f-7b57-4d74-a0a3-cf6f2e989e1c | cdh         | 5.11.0         |
| cdh-5110-default-nodemanager-datanode | c5247564-110c-4665-baa9-474995748e9b | cdh         | 5.11.0         |
| cdh-5110-default-master-additional    | d67ce419-2674-42aa-8c7e-802f56562815 | cdh         | 5.11.0         |
+---------------------------------------+--------------------------------------+-------------+----------------+
```

Lastly we need to create the CDH cluster template. Replace the node_group_template_ids with ones from your cloud.

```
$ cat cluster.json
{
    "plugin_name": "cdh",
    "hadoop_version": "5.11.0",
    "node_groups": [
        {
            "name": "worker-nodemanager-datanode",
            "count": 3,
            "node_group_template_id": "c5247564-110c-4665-baa9-474995748e9b"
        },
        {
            "name": "manager",
            "count": 1,
            "node_group_template_id": "3a82330f-7b57-4d74-a0a3-cf6f2e989e1c"
        },
        {
            "name": "master-core",
            "count": 1,
            "node_group_template_id": "315cd17a-4c70-4ec8-bc41-a5715edae708"
        },
        {
            "name": "master-additional",
            "count": 1,
            "node_group_template_id": "d67ce419-2674-42aa-8c7e-802f56562815"
        }
    ],
    "name": "cdh-5110-default-cluster",
    "cluster_configs": {
        "general": {
            "Timeout for Cloudera Manager starting": "600"
        }
    },
    "is_protected": false
}
$ openstack dataprocessing cluster template create --json cluster.json
```

Now, you are ready to launch your first cluster. This can be done with the CLI (like below), API or the Horizon (like in the video).

```
$ openstack dataprocessing cluster create --name cdh5110-test --cluster-template cdh-5110-default-cluster --image cdh5110 --user-keypair operator --neutron-network private
```

You can follow along with the status of the cluster as it deploys by watching the command below.

```
$ openstack dataprocessing cluster list
+--------------+--------------------------------------+-------------+----------------+-------------+
| Name         | Id                                   | Plugin name | Plugin version | Status      |
+--------------+--------------------------------------+-------------+----------------+-------------+
| cdh5110-test | 3ce340d1-e207-4a9a-a083-d74ceebc5964 | cdh         | 5.11.0         | Configuring |
+--------------+--------------------------------------+-------------+----------------+-------------+
```

Once the cluster is up, I saw that I needed to restart the Cloudera scm agent on each node to fix the clock offset health errors. ssh cloud-user@{IP of Node} ‘sudo systemctl restart cloudera-scm-agent’

There are also a few minor configuration entries that need to be put in. You can see those in the end of the video around 5:36 in.

These same concepts can be applied to other cluster packages like MapR, Apache Spark and Ambarri. However, you’d obvioulsy need to mod the templates and build the images for those plugins. [Sahara default templates](https://github.com/openstack/sahara/tree/master/sahara/plugins/default_templates) are available on GitHub.
