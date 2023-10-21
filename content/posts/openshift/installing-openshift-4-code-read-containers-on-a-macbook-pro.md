---
title: Installing OpenShift 4 Code Read Containers on a MacBook Pro
weight: 6
type: docs
prev: posts/openshift/openshift-4-3-bare-metal-lab
next: posts/openshift/troubleshooting-openshift-container-storage-4-x
sidebar:
  open: true
---

![Macbook Pros](macbooks.jpeg)

There is a ton of interest in learning Kubernetes and Red Hat's distribution of Kubernetes, OpenShift. One of the ways Red Hat makes available to experience OpenShift 4 is via [Code Ready Containers (CRC)](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/1.11/html-single/getting_started_guide/index). In this article I will walk through the steps to install CRC on a MacBook Pro. This same procedure should work on any system running macOS.

Note that CRC can also be installed on Linux or Windows systems.

For this write up, I am going to use the [ocp-install-demo repository](https://gitlab.com/redhatdemocentral/ocp-install-demo) out of [Red Hat Demo Central](https://gitlab.com/redhatdemocentral) on GitLab.com. CRC is already pretty easy, but this particular repo is aimed at simplifying some of the procedures across those 3 different operating systems.

You can see me go through this write up in the video below.

{{< youtube fYVioaEx2HY >}}

## Prerequisites

CodeReady Containers requires the following system resources:

* macOS 10.12 Sierra or newer
* 4 virtual CPUs (vCPUs)
* 9 GB of free memory
* 35 GB of storage space

You also need a Red Hat Network (RHN) account in order to access the OpenShift Cluster Manager on cloud.redhat.com. You will need a pull-secret from there later in the steps. RHN accounts are free, so if you don't have one, you just need to [register](https://www.redhat.com/wapps/ugc/register.html).

## Process

Open the terminal app to get started.

On macOS, we will use hyperkit as the hypervisor. I also installed wget to make it easy to download some of the needed artifacts.

```
kejones:~/Git$ brew install hyperkit
...
kejones:~/Git$ brew install wget
```

Now let's create a directory to work in and clone the ocp-install-demo git repository.

```
kejones:~/Git$ mkdir redhatdemocentral
kejones:~/Git$ cd redhatdemocentral
kejones:~/Git/redhatdemocentral/$ git clone https://gitlab.com/redhatdemocentral/ocp-install-demo.git
kejones:~/Git/redhatdemocentral/$ cd ocp-install-demo/
```

We can now run the init.sh script that is in the repository. The truth is that the script would check for hyperkit and let you know if you didn't have it installed. It also has checks for other items that are required.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ./init.sh
...
HyperKit is intalled...

OpenShift CLI tooling is required but not installed yet... download 4.4 here (unzip and put on your path): https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.6
```

See the failure there is telling me that I need to download, unpack and add the OpenShift CLI tools (oc and kubectl) to my path for finding executables. I can do this very simply using wget, tar and by making a few symbolic links.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.6/openshift-client-mac-4.4.6.tar.gz
kejones:~/Git/redhatdemocentral/ocp-install-demo$ tar xvf openshift-client-mac-4.4.6.tar.gz
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ln -s /Users/kejones/Git/redhatdemocentral/ocp-install-demo/oc /usr/local/bin/oc
kevin:~/Git/redhatdemocentral/ocp-install-demo$ ln -s /Users/kejones/Git/redhatdemocentral/ocp-install-demo/kubectl /usr/local/bin/kubectl
```

Now if we rerun the init.sh script, we will see that I have another artifact to download, unpack and add to my path.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ./init.sh
...
HyperKit is intalled...

OpenShift CLI tools installed... checking for valid version...

Version of installed OpenShift command line tools correct... 4.4

Code Ready Containers is not yet installed... download here (unzip and put on your path): https://mirror.openshift.com/pub/openshift-v4/clients/crc/latest/crc-macos-amd64.tar.xz
```

This time the message is telling me that I need to download the CRC package.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/latest/crc-macos-amd64.tar.xz
kejones:~/Git/redhatdemocentral/ocp-install-demo$ tar xvf crc-macos-amd64.tar.xz
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ln -s /Users/kevin/Git/redhatdemocentral/ocp-install-demo/crc-macos-1.11.0-amd64/crc /usr/local/bin/crc
```

Rerun the init.sh script again, we will see that I get further, but I need 1 last item. With OpenShift 4 installations, you need a pull secret. This pull secret is unique for your Red Hat Network account. Your pull secret file can be downloaded from cloud.redhat.com OpenShift Cluster Manager.

Visit [https://cloud.redhat.com/openshift/install/crc/installer-provisioned](https://cloud.redhat.com/openshift/install/crc/installer-provisioned) and click the "Download pull secret" button. Once it's download move it to the ocp-install-demo directory.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ mv ~/Downloads/pull-secret.txt ./
```

Now we need to do a quick edit to the init.sh script to point to the pull secret file. You can use whatever text editor you are comfortable with. I'm using vi here (and in the video). If you don't know vi, nano might be a better option.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ vi init.sh
...
# Uncomment and set to your PULL-SECRET file location and admin password.
SECRET_PATH=/Users/kejones/Git/redhatdemocentral/ocp-install-demo/pull-secret.txt
```

Once you save the file with the SECRET_PATH set to point to your pull-secret.txt file, we can rerun the init.sh. You could certainly take care of all of these 4 items without rerunning the script. I wanted to show how the script does some checks to make sure all of the pieces are in place on the system.

```
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ./init.sh
kejones:~/Git/redhatdemocentral/ocp-install-demo$ ./init.sh 


####################################################################
##                                                                ##
##  Setting up OpenShift Container Plaform locally with:          ##
##                                                                ##
##                                                                ##
##    ####  ###  ####  #####     ##### #####  ###  ####  #   #    ##
##   #     #   # #   # #         #   # #     #   # #   # #   #    ##
##   #     #   # #   # ###       ##### ###   ##### #   #  ###     ##
##   #     #   # #   # #         #  #  #     #   # #   #   #      ##
##    ####  ###  ####  #####     #   # ##### #   # ####    #      ##
##                                                                ##
##                                                                ##
##    ####  ###  #   # #####  ###  ##### #   # ##### ##### #####  ##
##   #     #   # ##  #   #   #   #   #   ##  # #     #   # #      ##
##   #     #   # # # #   #   #####   #   # # # ###   #####  ###   ##
##   #     #   # #  ##   #   #   #   #   #  ## #     #  #      #  ##
##    ####  ###  #   #   #   #   # ##### #   # ##### #   # #####  ##
##                                                                ##
##                                                                ##
##  https://gitlab.com/redhatdemocentral/ocp-install-demo         ##
##                                                                ##
####################################################################

HyperKit is installed...

OpenShift command line tools installed... checking for valid version...

Version of installed OpenShift command line tools correct... 4.4

Code Ready Container is installed on your OSX machine...

Running Code Ready Containers setup on this machine, even if done before...

INFO Checking if oc binary is cached              
INFO Checking if podman remote binary is cached   
INFO Checking if goodhosts binary is cached       
INFO Checking if CRC bundle is cached in '$HOME/.crc' 
INFO Checking if running as non-root              
INFO Checking if HyperKit is installed            
INFO Checking if crc-driver-hyperkit is installed 
INFO Checking file permissions for /etc/resolver/testing 
Setup is complete, you can now run 'crc start' to start the OpenShift cluster

Before starting, setting up pull secret file location...

Setting pull-secret-file in cofiguration to: /Users/kevin/Git/redhatdemocentral/ocp-install-demo/pull-secret.txt

Setting CPU count in cofiguration to: 4

Setting memory in cofiguration to: 14336

Starting Code Ready Containers platform...

This can take some time, so feel free to grab a coffee...

#####################################################################
##                                                                 ##
##   ####  ###  ##### ##### ##### #####   ##### ##### #   # #####  ##
##  #     #   # #     #     #     #         #     #   ## ## #      ##
##  #     #   # ####  ####  ###   ###       #     #   # # # ###    ##
##  #     #   # #     #     #     #         #     #   #   # #      ##
##   ####  ###  #     #     ##### #####     #   ##### #   # #####  ##
##                                                                 ##
#####################################################################


INFO Checking if oc binary is cached              
INFO Checking if podman remote binary is cached   
INFO Checking if goodhosts binary is cached       
INFO Checking if running as non-root              
INFO Checking if HyperKit is installed            
INFO Checking if crc-driver-hyperkit is installed 
INFO Checking file permissions for /etc/resolver/testing 
INFO Extracting bundle: crc_hyperkit_4.4.5.crcbundle ... 
INFO Checking size of the disk image /Users/kevin/.crc/cache/crc_hyperkit_4.4.5/crc.qcow2 ... 
INFO Creating CodeReady Containers VM for OpenShift 4.4.5... 
INFO CodeReady Containers VM is running           
INFO Verifying validity of the cluster certificates ... 
INFO Restarting the host network                  
INFO Check internal and public DNS query ...      
INFO Check DNS query from host ...                
INFO Generating new SSH key                       
INFO Copying kubeconfig file to instance dir ...  
INFO Starting OpenShift kubelet service           
INFO Configuring cluster for first start          
INFO Adding user's pull secret ...                
INFO Updating cluster ID ...                      
INFO Starting OpenShift cluster ... [waiting 3m]  
INFO                                              
INFO To access the cluster, first set up your environment by following 'crc oc-env' instructions 
INFO Then you can access it by running 'oc login -u developer -p developer https://api.crc.testing:6443' 
INFO To login as an admin, run 'oc login -u kubeadmin -p 8rynV-SeYLc-h8Ij7-YPYcz https://api.crc.testing:6443' 
INFO                                              
INFO You can now run 'crc console' and use these credentials to access the OpenShift web console 
Started the OpenShift cluster
WARN The cluster might report a degraded or error state. This is expected since several operators have been disabled to lower the resource usage. For more information, please consult the documentation 

Retrieving the admin password...

Retrieving oc client host login from kubeconfig file...

Set OCP_HOST to:   https://api.crc.testing:6443

Logging in as developer using oc client:

The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y
```

You need to answer y and hit enter to allow use of the insecure connection. Don't worry. CRC uses self-signed certificates, and this is a security check to let you know that the self-signed cert isn't trusted by the system. This is fine for our development use case.

Once the install is complete, you will see an output like below with instructions to connect.

```
======================================================
=                                                    =
=  Install complete, get ready to rock.              =
=                                                    =
=  The server is accessible via web console at:      =
=                                                    =
= https://console-openshift-console.apps-crc.testing =
=                                                    =
=  Log in as admin: kubeadmin                        =
=         password: 8rynV-SeYLc-h8Ij7-YPYcz          =
=                                                    =
=  Log in as dev: developer                          =
=       password: developer                          =
=                                                    =
=  Now get your Red Hat Demo Central example         =
=  projects here:                                    =
=                                                    =
=     https://github.com/redhatdemocentral           =
=                                                    =
=  To stop, restart, or delete your OCP cluster:     =
=                                                    =
=     $ crc {stop | start | delete}                  =
=                                                    =
======================================================
```

You can now just use the crc command line tool directly. You should check the status of your CRC instance to see if the VM is in running state and the OpenShift cluster is in running state.

You can run the crc console command to open a browser tab (using your default browser) to bring up the OpenShift console.

## Next Step

There are several repositories on Red Hat Demo Central to deploy on your new CRC OpenShift 4 environment.

Go to [https://gitlab.com/redhatdemocentral](https://gitlab.com/redhatdemocentral) and try some out.
