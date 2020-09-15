# AppLCM Helm Chart
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Deploy your own private MECM.

## How Do I Enable the Edgegallery Repository for Helm 2?
To add the Helm Edgegallery Charts for your local client, run `helm repo add`:

```
$ helm repo add edgegallery https://edgegallery.github.io/helm-charts
"edgegallery" has been added to your repositories
```

## Prerequisites
* Kubernetes 1.6+
* **Helm 2**
* [If enabled] nfs server and RW access to it
* [If enabled] nfs-client-provisioner for dynamic provisioning
```
helm install --name nfs-client-provisioner --set nfs.server=<nfs_sever_ip> --set nfs.path=<nfs_server_directory> stable/nfs-client-provisioner 
```
* [If enabled] nginx-ingress-controller for ingress
```
kubectl label node <node_name> node=edge
helm install --name nginx-ingress-controller stable/nginx-ingress --set controller.kind=DaemonSet --set controller.nodeSelector.node=edge --set controller.hostNetwork=true
```
## Configuration
By default this chart will not have persistent storage.
```

The following table lists common configurable parameters of the chart and
their default values. See values.yaml for all available options.

| Parameter                               | Description                                   | Default                  |
|-----------------------------------------|-----------------------------------------------|--------------------------|
| `image.repository`                      | Image repository                              | `edgegallery/mecm-meo`   |
| `image.tag`                             | Image tag                                     | `0.2`                    |
| `image.pullPolicy`                      | Image pullPolicy                              | `Always`                 |
| `persistence.enabled`                   | Whether to use persistent storage             | `false`                  |
| `expose.port`                           | Expose port                                   | `8091`                   |
| `expose.nodePort`                       | Expose nodePort                               | `30201`                  |
| `expose.port`                           | Expose port                                   | `8092`                   |
| `expose.nodePort`                       | Expose nodePort                               | `30202`                  |
| `expose.port`                           | Expose port                                   | `8093`                   |
| `expose.nodePort`                       | Expose nodePort                               | `30203`                  |
| `ssl.secretName`                        | SSL keystore secret name                      | ``                       |
| `mecm.secretName`                       | MECM secret name                              | ``                       |

Specify each parameter using the `--set key=value[,key=value]` argument to
`helm install`.
```

### JWT configuration
Edgegallery use jwt to implement authentication between center and edges, so need to config public key and private key 
for usermgmt,and config public key for mecm,**the public key of mecm should be the same as the public key of user-mgmt**.
```shell
## Generate a RSA keypair through openssl
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj /C=CN/ST=Peking/L=Beijing/O=edgegallery/CN=edgegallery -out ca.csr
openssl x509 -req -days 365 -in ca.csr -extfile OPENSSL_CNF_PATH -extensions v3_ca -signkey ca.key -out ca.crt

### SSL configuration
SSL enabled for mecm-meo, need to provide a keystore secret:
```shell
## Generate a keystore through keytool
keytool -genkey -alias edgegallery \
  -dname "CN=edgegallery,OU=edgegallery,O=edgegallery,L=edgegallery,ST=edgegallery,C=CN" \
  -storetype PKCS12 -keyalg RSA -keysize 2048 -storepass ***** -keystore keystore.p12 -validity 365
keytool -importcert -file ca.crt -alias clientkey -keystore keystore.jks -storepass *****

## Create a keystore secret
kubectl create secret generic edgegallery-mecm-ssl-secret \
  --from-file=keystore.p12=keystore.p12 \
  --from-file=keystore.jks=keystore.jks \
  --from-literal=keystorePassword=***** \
  --from-literal=truststorePassword=***** \
  --from-literal=keystoreType=PKCS12 \
  --from-literal=keyAlias=edgegallery

## Postgres init sql
Postgres init db example (postgres_init.sql)
CREATE USER user-name WITH PASSWORD '*****' CREATEDB;
CREATE DATABASE db-name
    WITH 
    OWNER = owner
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;
```

## Create a mecm-meo secret with postgres_init.sql file to create necessary db's
kubectl create secret generic edgegallery-mecm-secret \
  --from-file=postgres_init.sql=postgres_init.sql \
  --from-literal=postgresPassword=***** \
  --from-literal=postgresApmPassword=***** \
  --from-literal=postgresAppoPassword=***** \
  --from-literal=postgresInventoryPassword=***** \
  --from-literal=edgeRepoUserName=***** \
  --from-literal=edgeRepoPassword=*****

## Installation
helm install  mecm mecm-meo-1.0.1.tgz \
  --set ssl.secretName=edgegallery-mecm-ssl-secret \
  --set mecm.secretName=edgegallery-mecm-secret
```

helm install --name my-mecm-meo edgegallery/mecm-meo \
  --set ssl.secretName=edgegallery-mecm-ssl-secret \
  --set mecm.secretName=edgegallery-mecm-secret
```

### Example enable push image configuration
helm install --name my-mecm-meo edgegallery/mecm-meo \
  --set ssl.secretName=edgegallery-mecm-ssl-secret \
  --set mecm.secretName=edgegallery-mecm-secret \
  --set mecm.pushImage=true
```

### Example persistence configuration
```shell
helm install --name my-mecm-meo edgegallery/mecm-meo \
  --set ssl.secretName=edgegallery-mecm-ssl-secret \
  --set mecm.secretName=edgegallery-mecm-secret
  --set persistence.enabled=true
```

## Uninstall
By default, a deliberate uninstall will result in the persistent volume
claim being deleted.

```shell
helm delete my-mecm-meo --purge
```
