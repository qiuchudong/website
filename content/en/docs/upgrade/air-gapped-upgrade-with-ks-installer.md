---
title: "Air-Gapped Upgrade with ks-installer"
keywords: "Air-Gapped, upgrade, kubesphere, 3.2.1"
description: "Use ks-installer and offline package to upgrade KubeSphere."
linkTitle: "Air-Gapped Upgrade with ks-installer"
weight: 7500
---

ks-installer is recommended for users whose Kubernetes clusters were not set up by [KubeKey](../../installing-on-linux/introduction/kubekey/), but hosted by cloud vendors or created by themselves. This tutorial is for **upgrading KubeSphere only**. Cluster operators are responsible for upgrading Kubernetes beforehand.


## Prerequisites

- You need to have a KubeSphere cluster running v3.1.x. If your KubeSphere version is v3.0.0 or earlier, upgrade to v3.1.x first.
- Read [Release Notes for 3.2.1](../../release/release-v321/) carefully.
- Back up any important component beforehand.
- A Docker registry. You need to have a Harbor or other Docker registries. For more information, see [Prepare a Private Image Registry](../../installing-on-linux/introduction/air-gapped-installation/#step-2-prepare-a-private-image-registry).
- Supported Kubernetes versions of KubeSphere 3.2.1: v1.19.x, v1.20.x, v1.21.x, and v1.22.x (experimental).

## Step 1: Prepare Installation Images

As you install KubeSphere in an air-gapped environment, you need to prepare an image package containing all the necessary images in advance.

1. Download the image list file `images-list.txt` from a machine that has access to Internet through the following command:

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/images-list.txt
   ```

   {{< notice note >}}

   This file lists images under `##+modulename` based on different modules. You can add your own images to this file following the same rule. To view the complete file, see [Appendix](../../installing-on-linux/introduction/air-gapped-installation/#image-list-of-kubesphere-v310).

   {{</ notice >}} 

2. Download `offline-installation-tool.sh`. 

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/offline-installation-tool.sh
   ```

3. Make the `.sh` file executable.

   ```bash
   chmod +x offline-installation-tool.sh
   ```

4. You can execute the command `./offline-installation-tool.sh -h` to see how to use the script:

   ```bash
   root@master:/home/ubuntu# ./offline-installation-tool.sh -h
   Usage:
   
     ./offline-installation-tool.sh [-l IMAGES-LIST] [-d IMAGES-DIR] [-r PRIVATE-REGISTRY] [-v KUBERNETES-VERSION ]
   
   Description:
     -b                     : save kubernetes' binaries.
     -d IMAGES-DIR          : the dir of files (tar.gz) which generated by `docker save`. default: ./kubesphere-images
     -l IMAGES-LIST         : text file with list of images.
     -r PRIVATE-REGISTRY    : target private registry:port.
     -s                     : save model will be applied. Pull the images in the IMAGES-LIST and save images as a tar.gz file.
     -v KUBERNETES-VERSION  : download kubernetes' binaries. default: v1.17.9
     -h                     : usage message
   ```

5. Pull images in `offline-installation-tool.sh`.

   ```bash
   ./offline-installation-tool.sh -s -l images-list.txt -d ./kubesphere-images
   ```

   {{< notice note >}}

   You can choose to pull images as needed. For example, you can delete `##k8s-images` and related images under it in `images-list.text` if you already have a Kubernetes cluster.

   {{</ notice >}} 

## Step 2: Push Images to Your Private Registry

Transfer your packaged image file to your local machine and execute the following command to push it to the registry.

```bash
./offline-installation-tool.sh -l images-list.txt -d ./kubesphere-images -r dockerhub.kubekey.local
```

{{< notice note >}}

The domain name is `dockerhub.kubekey.local` in the command. Make sure you use your **own registry address**.

{{</ notice >}} 

## Step 3: Download ks-installer

Similar to installing KubeSphere on an existing Kubernetes cluster in an online environment, you also need to download `kubesphere-installer.yaml`.

1. Execute the following command to download ks-installer and transfer it to your machine that serves as the taskbox for installation.

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/kubesphere-installer.yaml
   ```
   
2. Verify that you have specified your private image registry in `spec.local_registry` in `cluster-configuration.yaml`. Note that if your existing cluster was installed in an air-gapped environment, you may already have this field specified. Otherwise, run the following command to edit `cluster-configuration.yaml` of your existing KubeSphere v3.1.x cluster and add the private image registry:

   ```
   kubectl edit cc -n kubesphere-system
   ```

   For example, `dockerhub.kubekey.local` is the registry address in this tutorial, then use it as the value of `.spec.local_registry` as below:

   ```yaml
   spec:
     persistence:
       storageClass: ""
     authentication:
       jwtSecret: ""
     local_registry: dockerhub.kubekey.local # Add this line manually; make sure you use your own registry address.
   ```

3. Save `cluster-configuration.yaml` after you finish editing it. Replace `ks-installer` with your **own registry address** with the following command:

   ```bash
   sed -i "s#^\s*image: kubesphere.*/ks-installer:.*#        image: dockerhub.kubekey.local/kubesphere/ks-installer:v3.1.0#" kubesphere-installer.yaml
   ```

   {{< notice warning >}}

   `dockerhub.kubekey.local` is the registry address in the command. Make sure you use your own registry address.

   {{</ notice >}}

## Step 4: Upgrade KubeSphere

Execute the following command after you make sure that all steps above are completed.

```bash
kubectl apply -f kubesphere-installer.yaml
```

## Step 5: Verify Installation

When the installation finishes, you can see the content as follows:

```bash
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.0.2:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     the "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are up and running.
  2. Please change the default password after login.

#####################################################
https://kubesphere.io             20xx-xx-xx xx:xx:xx
#####################################################
```

Now, you will be able to access the web console of KubeSphere through `http://{IP}:30880` with the default account and password `admin/P@88w0rd`.

{{< notice note >}}

To access the console, make sure port 30880 is opened in your security group.

{{</ notice >}}
