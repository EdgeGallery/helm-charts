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
| `image.repository`                      | Image repository                              | `edgegallery/mecm-mepm`  |
| `image.tag`                             | Image tag                                     | `0.2`                    |
| `image.pullPolicy`                      | Image pullPolicy                              | `Always`                 |
| `persistence.enabled`                   | Whether to use persistent storage             | `false`                  |
| `expose.port`                           | Expose port                                   | `8094`                   |
| `expose.nodePort`                       | Expose nodePort                               | `30204`                  |
| `expose.port`                           | Expose port                                   | `8095`                   |
| `expose.nodePort`                       | Expose nodePort                               | `30205`                  |
| `jwt.publicKeySecretName`               | JWT public key secret                         | ``                       |
| `ssl.secretName`                        | SSL keystore secret name                      | ``                       |
| `mepm.secretName`                       | MEPM secret name                              | ``                       |

Specify each parameter using the `--set key=value[,key=value]` argument to
`helm install`.
```

### JWT configuration
Edgegallery use jwt to implement authentication between center and edges, so need to config public key and private key 
for usermgmt,and config public key for mecm-lcm,**the public key of mecm-lcm should be the same as the public key of user-mgmt**.
```shell
## Generate a RSA keypair through openssl
openssl genrsa -out rsa_private_key.pem 2048
openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM -passout pass:***** -out encrypted_rsa_private_key.pem


## Create a jwt public key secret for mecm-mepm
kubectl create secret generic mecm-mepm-jwt-public-secret \
  --from-file=publicKey=rsa_public_key.pem

```

### SSL configuration
SSL is enabled for mecm-mepm, need to provide a server certificate, server key and root certificate:
```shell
## Generate ca certificate
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj /C=CN/ST=Peking/L=Beijing/O=edgegallery/CN=edgegallery -out ca.csr
openssl x509 -req -days 365 -in ca.csr -extfile OPENSSL_CNF_PATH -extensions v3_ca -signkey ca.key -out ca.crt
# generate tls certificate
openssl genrsa -out server_tls.key 2048
openssl rsa -in server_tls.key -aes256 -passout pass:***** -out server_encryptedtls.key

openssl req -new -key server_tls.key -subj /C=CN/ST=Beijing/L=Beijing/O=edgegallery/CN=edgegallery -out server_tls.csr
openssl x509 -req -in server_tls.csr -extfile OPENSSL_CNF_PATH -extensions v3_req -CA ca.crt -CAkey ca.key -CAcreateserial -out server_tls.crt

## Create a ssl secret
kubectl create secret generic mecm-mepm-ssl-secret \
  --from-file=server_tls.key=server_tls.key \
  --from-file=server_tls.crt=server_tls.crt \
  --from-file=ca.crt=ca.crt 

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

## Create a mecm-mepm secret with postgres_init.sql file to create necessary db's
kubectl create secret generic edgegallery-mepm-secret \
  --from-file=postgres_init.sql=postgres_init.sql \
  --from-literal=postgresPassword=***** \
  --from-literal=postgresLcmCntlrPassword=***** \
  --from-literal=postgresk8sPluginPassword=***** 


## Installation
helm install --name my-mecm-mepm edgegallery/mecm-mepm \
  --set jwt.publicKeySecretName=mecm-mepm-jwt-public-secret \
  --set ssl.secretName=mecm-mepm-ssl-secret \
  --set mepm.secretName=edgegallery-mepm-secret
```

### Example persistence configuration
```shell
helm install --name my-mecm-mepm edgegallery/mecm-mepm \
  --set jwt.publicKeySecretName=mecm-mepm-jwt-public-secret \
  --set ssl.secretName=mecm-mepm-ssl-secret \
  --set mepm.secretName=edgegallery-mepm-secret \
  --set persistence.enabled=true
```

## Uninstall
By default, a deliberate uninstall will result in the persistent volume
claim being deleted.

```shell
helm delete my-mecm-mepm --purge
```
