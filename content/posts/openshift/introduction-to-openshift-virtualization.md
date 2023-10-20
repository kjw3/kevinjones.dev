---
title: Introduction to OpenShift Virtulization
prev: posts/openshift/
sidebar:
  open: true
---

![Kubevirt Header Image](kubevirt-intro.png)

Recently I published the video below showing a very quick introduction to running virtual machines on [OpenShift](https://www.openshift.com/) with [KubeVirt](https://kubevirt.io/). One of the comments in response was a request to get a write up on deploying the KubeVirt UI shown in the video. I decided instead to post all of my steps for configuring KubeVirt and the UI on top of my existing OpenShift cluster. Note that my cluster had bare metal app nodes.

{{< youtube kzcToqgd23A >}}

## Prerequisites

Prior to configuring Kubevirt, you need a Kubernetes, OKD, or Openshift Container Platform cluster deployed. Those deployment instructions are well covered outside of this article. For the video, I am using an OKD 3.11 cluster with 1 master node, 1 infra node (both in VMs) and 8 bare metal app nodes.

If your app nodes are virtual machines, you will either need nested virt enabled on your hypervisors, or you can try [software emulation](https://github.com/kubevirt/kubevirt/blob/master/docs/software-emulation.md).

Access to a machine that has the oc client installed and logged in with cluster-admin privs.

## Configuring KubeVirt

Keep in mind that these are the steps I took on the cluster shown in the video. If you have suggestions for improvement, reach out.

First we set a few policies on our OpenShift cluster.

```
$ oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-privileged
$ oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-controller
$ oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-apiserver
```

Next, we apply the template for KubeVirt.

```
$ RELEASE=v0.11.0
$ oc apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt.yaml
```

Now let’s add the KubeVirt UI.

```
$ oc new-project kubevirt-web-ui
$ oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:kubevirt-web-ui:default
$ oc apply -f https://raw.githubusercontent.com/xsgordon/kubevirt-minishift-demo/master/kubevirt-web-ui.yaml
```

The KubeVirt CLI is currently still stand alone. So we need to download it and make it executable. You could also add it to your path.

```
$ curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/$RELEASE/virtctl-$RELEASE-linux-amd64
$ chmod +x virtctl
```

Last of all, let’s test a Virtual Machine! This template uses a small test linux image named [Cirros](http://download.cirros-cloud.net/).

```
$ oc create -f https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
$ ./virtctl start testvm
```

## Resources

1. [OpenShift Documentation](https://docs.openshift.com/)
2. [KubeVirt Docs](https://kubevirt.io/user-guide/docs/latest/welcome/index.html)
3. [Free OpenShift Labs](https://learn.openshift.com/)