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
## Configuration
By default this chart will not have persistent storage.
```

The following table lists common configurable parameters of the chart and
their default values. See values.yaml for all available options.

| Parameter                                      | Description                                   | Default                  |
|------------------------------------------------|-----------------------------------------------|--------------------------|
| `global.persistence.enabled`                   | Whether to enable persistent storage          | `false`                  |
| `global.persistence.storageClassName`          | Persistent storage className                  | `nfs-client`             |
| `global.ingress.enabled`                       | Whether to enable ingress                     | `false`                  |
| `global.ingress.tls.enabled`                   | Whether to enable ingress tls                 | `false`                  |
| `global.ingress.tls.secretName`                | Ingress secret name                           | ``                       |
| `global.ingress.hosts.auth`                    | Auth server domain name                       | ``                       |
| `global.ingress.hosts.appstore`                | Appstore domain name                          | ``                       |
| `global.ingress.hosts.developer`               | Developer domain name                         | ``                       |
| `global.ingress.hosts.mecm`                    | MECM domain name                              | ``                       |
| `global.ssl.enabled`                           | Whether to enable ssl for internal service    | `false`                  |
| `global.ssl.secretName`                        | Internal service ssl secret name              | ``                       |
| `global.oauth2.authServerAddress`              | Auth server access address **required**       | ``                       |
| `global.oauth2.clients.appstore.clientUrl`     | Appstore access address **required**          | ``                       |
| `global.oauth2.clients.developer.clientUrl`    | Developer access address **required**         | ``                       |
| `global.oauth2.clients.mecm.clientUrl`         | MECM access address **required**              | ``                       |
| `usermgmt.jwt.secretName`                      | Auth server jwt secret name                   | ``                       |
| `usermgmt.postgres.username`                   | Auth server postgres database username        | `usermgmt`               |
| `usermgmt.postgres.password`                   | Auth server postgres database password        | `********`             |
| `usermgmt.expose.nodePort`                     | Auth server expose NodePort                   | `30067`                  |
| `usermgmt.images.usermgmt.tag`                 | Auth server image tag                         | `0.2`                 |
| `appstore.postgres.username`                   | Appstore postgres database username           | `appstore`               |
| `appstore.postgres.password`                   | Appstore postgres database password           | `********`             |
| `appstore.expose.appstoreFe.nodePort`          | Appstore front end expose NodePort            | `30091`                  |
| `appstore.images.appstoreFe.tag`               | Appstore front end image tag                  | `0.2`                 |
| `appstore.images.appstoreBe.tag`               | Appstore back end image tag                   | `0.2`                 |
| `developer.postgres.username`                  | Developer postgres database username          | `developer`              |
| `developer.postgres.password`                  | Developer postgres database password          | `********`             |
| `developer.expose.developerFe.nodePort`        | Developer front end expose NodePort           | `30092`                  |
| `developer.images.developerFe.tag`             | Developer front end image tag                 | `0.2`                 |
| `developer.images.developerBe.tag`             | Developer back end image tag                  | `0.2`                 |
| `mecm.expose.mecmFe.nodePort`                  | MECM front end expose NodePort                | `30093`                  |
| `mecm.images.mecmFe.tag`                       | MECM front end image tag                      | `0.2`                 |
| `mecm.images.apiHandlerInfra.tag`              | MECM apiHandlerInfra image tag                | `0.2`                 |
| `mecm.images.meoBpmnInfra.tag`                 | MECM meoBpmnInfra image tag                   | `0.2`                 |
| `mecm.images.meoCatalogDbAdapter.tag`          | MECM meoCatalogDbAdapter image tag            | `0.2`                 |
| `mecm.images.mecmEsr.tag`                      | MECM mecmEsr image tag                        | `0.2`                 |
| `mecm.images.meoRequestDbAdapter.tag`          | MECM meoRequestDbAdapter image tag            | `0.2`                 |
| `mecm.images.mecmCatalog.tag`                  | MECM mecmCatalog image tag                    | `0.2`                 |
| `servicecenter.images.tag`                     | Service center mecmCatalog image tag          | `0.2`                 |

Specify each parameter using the `--set key=value[,key=value]` argument to
`helm install`.
```

### JWT configuration
Edgegallery use jwt to implement authentication between center and edges, so need to config public key and private key 
for usermgmt,and config public key for applcm,**the public key of applcm should be the same as the public key of user-mgmt**.
```shell
## Generate a RSA keypair through openssl
openssl genrsa -out rsa_private_key.pem 2048
openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM -passout pass:******** -out encrypted_rsa_private_key.pem

## Create a jwt secret for usermgmt
kubectl create secret generic user-mgmt-jwt-secret \
  --from-file=publicKey=rsa_public_key.pem \
  --from-file=encryptedPrivateKey=encrypted_rsa_private_key.pem \
  --from-literal=encryptPassword=********

## Create a jwt public key secret for applcm
kubectl create secret generic applcm-jwt-public-secret \
  --from-file=publicKey=rsa_public_key.pem

## Installation
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=http://<ip>:30067 \
  --set global.oauth2.clients.appstore.clientUrl=http://<ip>:30091 \
  --set global.oauth2.clients.developer.clientUrl=http://<ip>:30092 \
  --set global.oauth2.clients.mecm.clientUrl=http://<ip>:30093 \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret
```

### Ingress configuration
Edgegallery use nodeport to expose UI by default,If you want to use ingress to access UI, need to provide four domain names:
```shell
## Installation
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=http://<auth-server-domain-name> \
  --set global.oauth2.clients.appstore.clientUrl=http://<appstore-domain-name> \
  --set global.oauth2.clients.developer.clientUrl=http://<developer-domain-name> \
  --set global.oauth2.clients.mecm.clientUrl=http://<mecm-domain-name> \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret \
  --set global.ingress.enabled=true \
  --set global.ingress.hosts.auth=<auth-server-domain-name> \
  --set global.ingress.hosts.appstore=<appstore-domain-name> \
  --set global.ingress.hosts.developer=<developer-domain-name> \
  --set global.ingress.hosts.mecm=<mecm-domain-name>
```
If you want to enable ssl for ingress,need to provide certificates,and create a tls secret:
```shell
## Generate ca certificate
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj /C=CN/ST=Beijing/L=Biejing/O=edgegallery/CN=edgegallery.org -out ca.csr
openssl x509 -req -days 365 -in ca.csr -extensions v3_ca -signkey ca.key -out ca.crt

## Generate tls certificate
openssl genrsa -out tls.key 2048
openssl rsa -in tls.key -aes256 -passout pass:******** -out encryptedtls.key
openssl req -new -key tls.key -subj /C=CN/ST=Beijing/L=Biejing/O=edgegallery/CN=edgegallery.org -out tls.csr
openssl x509 -req -in tls.csr -extensions v3_usr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt

## Create a tls secret for ingress
kubectl create secret generic edgegallery-ingress-secret \
  --from-file=ca.crt=ca.crt \
  --from-file=tls.crt=tls.crt \
  --from-file=tls.key=tls.key

## Installation
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=https://<auth-server-domain-name> \
  --set global.oauth2.clients.appstore.clientUrl=https://<appstore-domain-name> \
  --set global.oauth2.clients.developer.clientUrl=https://<developer-domain-name> \
  --set global.oauth2.clients.mecm.clientUrl=http://<mecm-domain-name> \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret \
  --set global.ingress.enabled=true \
  --set global.ingress.hosts.auth=<auth-server-domain-name> \
  --set global.ingress.hosts.appstore=<appstore-domain-name> \
  --set global.ingress.hosts.developer=<developer-domain-name> \
  --set global.ingress.hosts.mecm=<mecm-domain-name> \
  --set global.ingress.tls.enabled=true \
  --set global.ingress.tls.secretName=edgegallery-ingress-secret
```

### SSL configuration
If you want to enable SSL of internal service, need to provide a keystore secret:
```shell
## Generate a keystore.p12 through keytool
keytool -genkey -alias edgegallery \
  -dname "CN=edgegallery,OU=edgegallery,O=edgegallery,L=edgegallery,ST=edgegallery,C=CN" \
  -storetype PKCS12 -keyalg RSA -keysize 2048 -storepass ******** -keystore keystore.p12 -validity 365

## Create a keystore secret
kubectl create secret generic edgegallery-ssl-secret \
  --from-file=keystore.p12=keystore.p12 \
  --from-literal=keystorePassword=******** \
  --from-literal=keystoreType=PKCS12 \
  --from-literal=keyAlias=edgegallery \
  --from-file=trust.cer=ca.crt \
  --from-file=server.cer=tls.crt \
  --from-file=server_key.pem=encryptedtls.key \
  --from-literal=cert_pwd=********

## Installation
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=https://<ip>:30067 \
  --set global.oauth2.clients.appstore.clientUrl=https://<ip>:30091 \
  --set global.oauth2.clients.developer.clientUrl=https://<ip>:30092 \
  --set global.oauth2.clients.mecm.clientUrl=https://<ip>:30093 \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret \
  --set global.ssl.enabled=true \
  --set global.ssl.secret=edgegallery-ssl-secret
```

### Example NodePort configuration
If you want to change expose nodePort,you can install edgegallery like this:
```shell
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=http://<ip>:<usermgmt-nodeport> \
  --set global.oauth2.clients.appstore.clientUrl=http://<ip>:<appstore-nodeport> \
  --set global.oauth2.clients.developer.clientUrl=http://<ip>:<developer-nodeport> \
  --set global.oauth2.clients.mecm.clientUrl=http://<ip>:<mecm-nodeport> \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret \
  --set usermgmt.expose.nodePort=<usermgmt-nodeport> \
  --set appstore.expose.appstoreFe.nodePort=<appstore-nodeport> \
  --set developer.expose.developerFe.nodePort=<developer-nodeport> \
  --set mecm.expose.mecmFe.nodePort=<mecm-nodeport>
```

### Example persistence configuration
```shell
helm install my-edgegallery edgegallery/edgegallery \
  --set global.oauth2.authServerAddress=https://<ip>:30067 \
  --set global.oauth2.clients.appstore.clientUrl=https://<ip>:30091 \
  --set global.oauth2.clients.developer.clientUrl=https://<ip>:30092 \
  --set global.oauth2.clients.mecm.clientUrl=https://<ip>:30093 \
  --set usermgmt.jwt.secretName=user-mgmt-jwt-secret \
  --set global.persistence.enabled=true
```
By default persistence storage will use "nfs-client" as storageClassName, you can config global.persistence.storageClassName
to change storageClassName.

## Uninstall
By default, a deliberate uninstall will result in the persistent volume
claim being deleted.
```shell
helm delete my-edgegallery
```
