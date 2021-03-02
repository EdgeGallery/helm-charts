# MEP Helm Chart
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Deploy MEP.

## Prepare ssl certs
```
  set +o history
  set +x

  export PG_ADMIN_PWD=admin-Pass123
  export KONG_PG_PWD=kong-Pass123
  export CERT_PWD=te9Fmv%qaq

  export MEP_CERTS_DIR=/tmp/.mep_tmp_cer
  export CERT_NAME=${CERT_NAME:-mepserver}
  export DOMAIN_NAME=edgegallery

  rm -rf ${MEP_CERTS_DIR}
  mkdir -p ${MEP_CERTS_DIR}
  cd ${MEP_CERTS_DIR} || exit

  generate ca certificate:
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -subj /C=CN/ST=Peking/L=Beijing/O=edgegallery/CN=${DOMAIN_NAME} -out ca.csr
  openssl x509 -req -days 365 -in ca.csr -extensions v3_ca -signkey ca.key -out ca.crt

  generate tls certificate:
  openssl genrsa -out ${CERT_NAME}_tls.key 2048
  openssl rsa -in ${CERT_NAME}_tls.key -aes256 -passout pass:${CERT_PWD} -out ${CERT_NAME}_encryptedtls.key

  echo -n ${CERT_PWD} > ${CERT_NAME}_cert_pwd

  openssl req -new -key ${CERT_NAME}_tls.key -subj /C=CN/ST=Beijing/L=Beijing/O=edgegallery/CN=${DOMAIN_NAME} -out ${CERT_NAME}_tls.csr
  openssl x509 -req -in ${CERT_NAME}_tls.csr -extensions v3_req -CA ca.crt -CAkey ca.key -CAcreateserial -out ${CERT_NAME}_tls.crt

  generate jwt public private key:
  openssl genrsa -out jwt_privatekey 2048
  openssl rsa -in jwt_privatekey -pubout -out jwt_publickey
  openssl rsa -in jwt_privatekey -aes256 -passout pass:${CERT_PWD} -out jwt_encrypted_privatekey

  remove unnecessary key file:
  rm ca.key
  rm ca.csr
  rm ca.crl
  rm ${CERT_NAME}_tls.csr
  rm jwt_privatekey

  setup read permission:
  cd ..
  chmod 600 ${MEP_CERTS_DIR}/*

  kubectl create ns mep

  kubectl -n mep create secret generic pg-secret \
    --from-literal=pg_admin_pwd=$PG_ADMIN_PWD \
    --from-literal=kong_pg_pwd=$KONG_PG_PWD \
    --from-file=server.key=${MEP_CERTS_DIR}/${CERT_NAME}_tls.key \
    --from-file=server.crt=${MEP_CERTS_DIR}/${CERT_NAME}_tls.crt

  kubectl -n mep create secret generic mep-ssl \
    --from-literal=cert_pwd=$CERT_PWD \
    --from-file=server.cer=${MEP_CERTS_DIR}/${CERT_NAME}_tls.crt \
    --from-file=server_key.pem=${MEP_CERTS_DIR}/${CERT_NAME}_encryptedtls.key \
    --from-file=trust.cer=${MEP_CERTS_DIR}/ca.crt

  kubectl -n mep create secret generic mepauth-secret \
    --from-file=server.crt=${MEP_CERTS_DIR}/${CERT_NAME}_tls.crt \
    --from-file=server.key=${MEP_CERTS_DIR}/${CERT_NAME}_tls.key \
    --from-file=ca.crt=${MEP_CERTS_DIR}/ca.crt \
    --from-file=jwt_publickey=${MEP_CERTS_DIR}/jwt_publickey \
    --from-file=jwt_encrypted_privatekey=${MEP_CERTS_DIR}/jwt_encrypted_privatekey

  rm -rf ${MEP_CERTS_DIR}
  set -o history
```

## Setup dns metallb
```
   ## Get following yaml files from https://gitee.com/edgegallery/platform-mgmt/tree/master/offline/conf/edge/metallb
   kubectl apply -f namespace.yaml
   kubectl apply -f metallb.yaml
   kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
   sed -i "s/192.168.100.120/${EG_NODE_IP}/g" config-map.yaml
   kubectl apply -f config-map.yaml
```

## Setup network isolation
```
  ## Get following yaml files from https://gitee.com/edgegallery/platform-mgmt/tree/master/offline/conf/edge/network-isolation
  kubectl apply -f multus.yaml
  kubectl apply -f eg-sp-rbac.yaml
  kubectl apply -f eg-sp-controller.yaml

  ## Network Isolation on multinode (only x86 now)
  kubectl apply -f whereabouts-daemonset-install.yaml
  kubectl apply -f whereabouts.cni.cncf.io_ippools.yaml
```

## Setup interfaces
```
  ## Same interface value can also be set to MP1 and MM5:
  export $EG_NODE_EDGE_MP1=interface-1
  export $EG_NODE_EDGE_MM5=interface-2

  ## These ip's range is betwen x.1.1.2~x.1.1.24 . Rest reserved for allocation to pods. ip_prefix is starts with value 2. Increment it for every node to keep ip unique.
  ip link add vxlan-mp1 type vxlan id 100 group 239.1.1.1 dstport 4789 dev $EG_NODE_EDGE_MP1
  ip link set vxlan-mp1 up

  ip link add vxlan-mm5 type vxlan id 200 group 239.1.1.1 dstport 4789 dev $EG_NODE_EDGE_MM5
  ip link set vxlan-mm5 up

  ip link add eg-mp1 link vxlan-mp1 type macvlan mode bridge
  ip addr add 200.1.1.$ip_prefix/24 dev eg-mp1
  ip link set dev eg-mp1 up

  ip link add eg-mm5 link vxlan-mm5 type macvlan mode bridge
  ip addr add 100.1.1.$ip_prefix/24 dev eg-mm5
  ip link set dev eg-mm5 up
```

## Add helm repo
```
helm repo add eg http://helm.edgegallery.org:30002/chartrepo/edgegallery_helm_chart
```

## Install MEP helm-chart
```
 ## Aio mode
  helm install mep-edgegallery eg/mep \
  --set networkIsolation.ipamType=host-local \
  --set networkIsolation.phyInterface.mp1=interface-2 \
  --set networkIsolation.phyInterface.mm5=interface-1
 ## muno mode
  helm install mep-edgegallery eg/mep \
  --set networkIsolation.ipamType=whereabouts \
  --set networkIsolation.phyInterface.mp1=interface-2 \
  --set networkIsolation.phyInterface.mm5=interface-1
```

## Uninstall
```
  helm uninstall mep-edgegallery

  ## Remove ssl config:
  kubectl delete secret pg-secret -n mep
  kubectl delete secret mep-ssl -n mep
  kubectl delete secret mepauth-secret -n mep
  kubectl delete ns mep
  
  ## Cleanup netwok isolation setup:
  kubectl delete -f eg-sp-controller.yaml
  kubectl delete -f eg-sp-rbac.yaml
  kubectl delete -f multus.yaml
  kubectl delete -f whereabouts-daemonset-install.yaml
  kubectl delete -f whereabouts.cni.cncf.io_ippools.yaml
  rm /opt/cni/bin/macvlan /opt/cni/bin/host-local
  ip link set dev eg-mp1 down
  ip link delete eg-mp1
  ip link set dev eg-mm5 down
  ip link delete eg-mm5
  ip link delete vxlan-mp1
  ip link delete vxlan-mm5
  rm /opt/cni/bin/multus

  ## Cleanup dns setup:
  kubectl delete -f config-map.yaml
  kubectl delete -f metallb.yaml
  kubectl delete -f namespace.yaml
  kubectl delete secret memberlist -n metallb-system
```
