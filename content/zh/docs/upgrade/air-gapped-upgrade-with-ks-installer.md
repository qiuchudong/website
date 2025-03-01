---
title: "使用 ks-installer 离线升级"
keywords: "离线环境, 升级, kubesphere, 3.2.1"
description: "使用 ks-installer 和离线包升级 KubeSphere。"
linkTitle: "使用 ks-installer 离线升级"
weight: 7500
---

对于 Kubernetes 集群不是通过 [KubeKey](../../installing-on-linux/introduction/kubekey/) 部署而是由云厂商托管或自行部署的用户，推荐使用 ks-installer。本教程**只用于升级 KubeSphere**。集群运维员应负责提前升级 Kubernetes。


## 准备工作

- 您需要有一个运行 KubeSphere v3.1.x 的集群。如果您的 KubeSphere 是 v3.0.0 或更早的版本，请先升级至 v3.1.x。
- 请仔细阅读 [3.2.1 版本说明](../../release/release-v321/)。
- 提前备份所有重要的组件。
- Docker 仓库。您需要有一个 Harbor 或其他 Docker 仓库。有关更多信息，请参见[准备一个私有镜像仓库](../../installing-on-linux/introduction/air-gapped-installation/#步骤-2准备一个私有镜像仓库)。
- KubeSphere 3.2.1 支持的 Kubernetes 版本：v1.19.x、v1.20.x、v1.21.x 和 v1.22.x（实验性支持）。

## 步骤 1：准备安装镜像

当您在离线环境中安装 KubeSphere 时，需要事先准备一个包含所有必需镜像的镜像包。

1. 使用以下命令从能够访问互联网的机器上下载镜像清单文件 `images-list.txt`：

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/images-list.txt
   ```

   {{< notice note >}}

   该文件根据不同的模块列出了 `##+modulename` 下的镜像。您可以按照相同的规则把自己的镜像添加到这个文件中。要查看完整文件，请参见[附录](../../installing-on-kubernetes/on-prem-kubernetes/install-ks-on-linux-airgapped/#kubesphere-v321-镜像清单)。

   {{</ notice >}} 

2. 下载 `offline-installation-tool.sh`。

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/offline-installation-tool.sh
   ```

3. 使 `.sh` 文件可执行。

   ```bash
   chmod +x offline-installation-tool.sh
   ```

4. 您可以执行命令 `./offline-installation-tool.sh -h` 来查看如何使用脚本：

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

5. 在 `offline-installation-tool.sh` 中拉取镜像。

   ```bash
   ./offline-installation-tool.sh -s -l images-list.txt -d ./kubesphere-images
   ```

   {{< notice note >}}

   您可以根据需要选择拉取的镜像。例如，如果已经有一个 Kubernetes 集群了，您可以在 `images-list.text` 中删除 `##k8s-images` 和在它下面的相关镜像。

   {{</ notice >}} 

## 步骤 2：推送镜像至您的私有仓库

将打包的镜像文件传输至您的本地机器，并运行以下命令把它推送至仓库。

```bash
./offline-installation-tool.sh -l images-list.txt -d ./kubesphere-images -r dockerhub.kubekey.local
```

{{< notice note >}}

命令中的域名是 `dockerhub.kubekey.local`。请确保使用您**自己仓库的地址**。

{{</ notice >}} 

## 步骤 3：下载 ks-installer

与在现有 Kubernetes 集群上在线安装 KubeSphere 相似，您需要事先下载 `kubesphere-installer.yaml`。

1. 执行以下命令下载 ks-installer，并将其传输至您充当任务机的机器，用于安装。

   ```bash
   curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/kubesphere-installer.yaml
   ```
   
2. 验证您已在 `cluster-configuration.yaml` 中的 `spec.local_registry` 字段指定了私有镜像仓库地址。请注意，如果您的已有集群通过离线安装方式搭建，您应该已配置了此地址。如果您的集群采用在线安装方式搭建而需要进行离线升级，执行以下命令编辑您已有 KubeSphere v3.1.x 集群的 `cluster-configuration.yaml` 文件，并添加私有镜像仓库地址：

   ```bash
   kubectl edit cc -n kubesphere-system
   ```

   例如，本教程中的仓库地址是 `dockerhub.kubekey.local`，将它用作 `.spec.local_registry` 的值，如下所示：

   ```yaml
   spec:
     persistence:
       storageClass: ""
     authentication:
       jwtSecret: ""
     local_registry: dockerhub.kubekey.local # Add this line manually; make sure you use your own registry address.
   ```

3. 编辑完成后保存 `cluster-configuration.yaml`。使用以下命令将 `ks-installer` 替换为您**自己仓库的地址**。

   ```bash
   sed -i "s#^\s*image: kubesphere.*/ks-installer:.*#        image: dockerhub.kubekey.local/kubesphere/ks-installer:v3.1.0#" kubesphere-installer.yaml
   ```

   {{< notice warning >}}

   命令中的仓库地址是 `dockerhub.kubekey.local`。请确保使用您自己仓库的地址。

   {{</ notice >}}

## 步骤 4：升级 KubeSphere

确保上述所有步骤都已完成后，执行以下命令。

```bash
kubectl apply -f kubesphere-installer.yaml
```

## 步骤 5：验证安装

安装完成后，您会看到以下内容：

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

现在，您可以通过 `http://{IP}:30880` 使用默认帐户和密码 `admin/P@88w0rd` 访问 KubeSphere 的 Web 控制台。

{{< notice note >}}

要访问控制台，请确保在您的安全组中打开端口 30880。

{{</ notice >}}
