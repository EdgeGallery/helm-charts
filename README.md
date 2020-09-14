# Edgegallery Helm Chart
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Deploy your own private Edgegallery.

## How Do I Enable the Edgegallery Repository for Helm 3?

To add the Helm Edgegallery Charts for your local client, run `helm repo add`:

```
$ helm repo add edgegallery https://edgegallery.github.io/helm-charts
"edgegallery" has been added to your repositories
```

## Prerequisites
* Kubernetes 1.6+
* Helm 3+
* [If enabled] nfs server and RW access to it
* [If enabled] nfs-client-provisioner for dynamic provisioning
```
helm install nfs-client-provisioner --set nfs.server=<nfs_sever_ip> --set nfs.path=<nfs_server_directory> stable/nfs-client-provisioner 
```
* [If enabled] nginx-ingress-controller for ingress
```
kubectl label node <node_name> node=edge
helm install nginx-ingress-controller stable/nginx-ingress --set controller.kind=DaemonSet --set controller.nodeSelector.node=edge --set controller.hostNetwork=true
```
## Edgegallery Installation
* Center Node: [Edgegallery](https://github.com/EdgeGallery/helm-charts/tree/master/edgegallery)

* Edge Node: [AppLCM](https://github.com/EdgeGallery/helm-charts/tree/master/applcm)
