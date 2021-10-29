# Kubernetes Backup e Disaster Recovery 
# Kasten's K10 Platform

A Kasten é uma empresa independente focada em soluções para plataforma Kubernetes, adquirida pela Veeam Backup recentemente, é líder premiado em Backup e recuperação de desastres do Kuberentes.

Kasten K10 é uma plataforma de gerenciamento de dados desenvolvida especialmente para kubernetes, plataforma amigável, escalonável  e seguro para backup e restauração de aplicações e mobilidade.

# Considerações:

A solução se demonstra estável e de fácil gerenciamento, para backup e restore no cluster de origem é provável que seja menos aplicável caso você tenha pipelines e versionamento, sua utilização será de grande valor em casos de incidente relacionado a perda de dados acidental ou maliciosa e também na falha do cluster (infraestrutura) do Kuberentes.

 Casos de Uso:
   - Perda de dados
   - Falha do Cluster/Infraestrutura/Hardware
   - Proteção das Aplicações em Cluster Externo / Disaster Recovery
   - Migração entre provedores de Cloud
   - MultiCloud Deployment

 Documentação:
   - Instalação do Helm: https://helm.sh/docs/intro/install/
   - Pré-requisitos para instalação Kasten K10: https://docs.kasten.io/latest/install/requirements.html
   - Doc. Oficial da plataforma: https://docs.kasten.io/latest/index.html

# Topologia

![K10 Tecnologia](https://user-images.githubusercontent.com/52961166/139346555-bbecdf44-820e-4a63-af0f-5b12017516a2.JPG)

# Integrações e Extensibilidade

![K10](https://user-images.githubusercontent.com/52961166/139346686-a854bf72-cab0-4967-b7b7-a347e6aa03e8.JPG)

Bom galera, era isso que eu tinha para vocês hoje, espero que gostem do conteúdo, deixe seu comentário e compartilhe com os amigos.

# O CSI snapshotter faz parte da implementação do Kubernetes de Container Storage Interface (CSI).

https://github.com/kubernetes-csi/external-snapshotter

Instalar the CSI snapshotter

Aplicar VolumeSnapshot CRDs
```sh
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```
Faça isso uma vez por cluster 

Instalar Common Snapshot Controller:

Atualize o namespace para um valor apropriado para seu ambiente (por exemplo, kube-system) 
```sh
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```
Faça isso uma vez por cluster 

# Instalar CSI Driver:

Siga as instruções fornecidas pelo fornecedor do driver CSI. 
```sh
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/csi-snapshotter/rbac-csi-snapshotter.yaml
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/csi-snapshotter/rbac-external-provisioner.yaml
kubectl apply -f https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/csi-snapshotter/setup-csi-snapshotter.yaml
```
Aqui está um exemplo para instalar o driver hostpath CSI de amostra 
```sh
git clone https://github.com/kubernetes-csi/csi-driver-host-path.git
cd csi-driver-host-path
```
Escola a versão do seu Kubernetes e faça o deploy 
```sh
./deploy/kubernetes-x.xx/deploy.sh
```
# Verificar requisitos para instalação

curl https://docs.kasten.io/tools/k10_primer.sh | bash

Adicionar repositorio Helm do Kastetn.io

helm repo add kasten https://charts.kasten.io/

Criar arquivo de anotações values.yaml

```sh  
nano values.yaml

apigateway:
  serviceResolver: endpoint
auth:
  tokenAuth:
    enabled: true
global:
  persistence:
    storageClass: managed-nfs-storage
grafana:
  persistence:
    storageClass: managed-nfs-storage
ingress:
  class: traefik
  create: true
  host: k10.<dominio.com.br>
  tls:
    enabled: true
    secretName: traefik-cert
  urlPath: /k10
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: default
prometheus:
  server:
    persistentVolume:
      storageClass: managed-nfs-storage
    securityContext:
      fsGroup: 0
      runAsGroup: 0
      runAsNonRoot: false
      runAsUser: 0
services:
  securityContext:
    fsGroup: 0
    runAsUser: 0
```
# Instalar K10 Kasten

```sh  
kubectl create namespace kasten-io; \
  helm install k10 kasten/k10 --namespace=kasten-io -f values.yaml
```
Depois de alguns minutos, podemos verificar se todos os pods estão funcionando.

```sh  
watch -n 2 "kubectl -n kasten-io get pods"  
```
# Acessar a Web Gui do K10

https://k10.<dominio.com.br>/k10/

Ok, por último, mas não menos importante, precisamos de um token para a autenticação baseada em tokens.
Para isso, precisamos criar uma conta de serviço e obter o token para essa conta do Kubernetes.
Este token pode ser copiado para nossa área de transferência e copiado para o campo de autenticação na próxima etapa.

```sh  
kubectl create serviceaccount kasten-sa --namespace kasten-io
kubectl create clusterrolebinding kasten-sa --clusterrole=cluster-admin --serviceaccount=kasten-io:kasten-sa
sa_secret=$(kubectl get serviceaccount kasten-sa -o jsonpath="{.secrets[0].name}" --namespace kasten-io)
kubectl get secret $sa_secret --namespace kasten-io -o jsonpath="{.data.token}{'\n'}" | base64 --decode
```
# Varias leituras de referencias que podem ajudar em uma melhor implementação.

https://veducate.co.uk/kasten-multi-cluster/
https://horstmann.in/how-i-built-my-kubernetes-homelab-part-6/
https://www.luizpessol.com.br/2021/07/28/backup-kubernetes-com-kasten-k10/
https://www.infracloud.io/blogs/k8s-disaster-recovery-using-kasten-k10/
https://horstmann.in/how-i-built-my-kubernetes-homelab-part-6/
https://www.unixarena.com/2021/09/kubernetes-backup-kasten-k10-test-drive.html/
https://veducate.co.uk/kasten-tanzu/


