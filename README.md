# Edgegallery 掌舵图
[![许可证](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

部署您自己的私人 Edgegallery。

## 如何为 Helm 3 启用 Edgegallery 存储库？

要为本地客户端添加 Helm Edgegallery 图表，请运行 `helm repo add`：

``
$ helm repo 添加 edgegallery http://helm.edgegallery.org:30002/chartrepo/edgegallery_helm_chart
“edgegallery”已添加到您的存储库中
``

## 先决条件
* Kubernetes 1.6+
* 掌舵 3+
* [如果启用] nfs 服务器和对它的 RW 访问
* [如果启用] nfs-client-provisioner 用于动态配置
``
helm install nfs-client-provisioner --set nfs.server=<nfs_sever_ip> --set nfs.path=<nfs_server_directory> stable/nfs-client-provisioner
``
* [如果启用] nginx-ingress-controller for ingress
``
kubectl 标签节点 <node_name> node=edge
helm install nginx-ingress-controller stable/nginx-ingress --set controller.kind=DaemonSet --set controller.nodeSelector.node=edge --set controller.hostNetwork=true
``
## Edgegallery 安装（离线）
* 安装指南：[链接](https://gitee.com/edgegallery/installer/blob/master/offline/README-cn.md)