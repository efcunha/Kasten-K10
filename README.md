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

Instalar the CSI snapshotter

https://github.com/efcunha/Kasten-K10/blob/main/csi-driver-host-path/README.md

# Verificar requisitos para instalação
```sh  
curl https://docs.kasten.io/tools/k10_primer.sh | bash
```
Em seguida, execute o seguinte comando com um StorageClass válido para implantar a ferramenta de pré-verificação:
```sh  
curl -s https://docs.kasten.io/tools/k10_primer.sh  | bash /dev/stdin -s ${STORAGE_CLASS}
```
Adicionar repositorio Helm do Kastetn.io
```sh  
helm repo add kasten https://charts.kasten.io/
```
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
    storageClassName: managed-nfs-storage
ingress:
  class: traefik
  create: true
  host: k10.dominio.com.br
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
  helm install k10 kasten/k10 --namespace=kasten-io -f values.yaml \
  --set injectKanisterSidecar.enabled=true \
  --set-string injectKanisterSidecar.objectSelector.matchLabels.component=db \
  --set-string injectKanisterSidecar.namespaceSelector.matchLabels.k10/injectKanisterSidecar=true
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
# Suponha que K10 esteja instalado no namespace 'kasten-io'

# Criar token usuario kasten-sa
kubectl create clusterrolebinding kasten-sa --clusterrole=cluster-admin --serviceaccount=kasten-io:kasten-sa

# obter o segredo SA
sa_secret=$(kubectl get serviceaccount kasten-sa -o jsonpath="{.secrets[0].name}" --namespace kasten-io)

# extrair o token
kubectl get secret $sa_secret --namespace kasten-io -o jsonpath="{.data.token}{'\n'}" | base64 --decode
```
# Local de armazenamento de arquivo NFS

Requisitos:

Um servidor NFS acessível a partir dos nós onde o K10 está instalado

Um compartilhamento NFS exportado, montado em todos os nós onde o K10 está instalado

Um volume persistente que define o compartilhamento NFS exportado semelhante ao exemplo abaixo: 
```sh
apiVersion: v1
kind: PersistentVolume
metadata:
   name: test-pv
spec:
   capacity:
      storage: 100Gi
   volumeMode: Filesystem
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   storageClassName: managed-nfs-storage
   mountOptions:
      - hard
      - nfsvers=4.1
   nfs:
      path: /opt/backup
      server: 172.16.0.22
``` 
Criar um Persistent Volume Claim com o mesmo nome de classe de armazenamento no namespace K10 (padrão kasten-io):
```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: test-pvc
   namespace: kasten-io
spec:
   storageClassName: managed-nfs-storage
   accessModes:
      - ReadWriteMany
   resources:
      requests:
         storage: 100Gi
``` 
Criar os volumes de armazenamento de arquivo no servidor NFS
```sh
kubectl apply -f k10-pv.yaml
kubectl apply -f k10-pvc.yaml
```
Uma vez que os requisitos acima sejam atendidos, um perfil de localização NFS FileStore pode ser criado na página de perfis usando o PVC criado acima. 

![location_profiles_nfs1](https://user-images.githubusercontent.com/52961166/140244175-9c02dc2c-5285-4cf3-82a9-fc21182a3e0c.png)

Quando Validar e Salvar for selecionado, o perfil de configuração será criado e um perfil semelhante ao seguinte aparecerá:

![location_profiles_example_nfs1](https://user-images.githubusercontent.com/52961166/140244284-95d3a8d1-8a73-4cd8-b829-a3b9b25ee2dc.png)

# Configurações de localização para migração

Se o perfil de localização for usado para exportar um aplicativo para migração entre clusters, ele será usado para armazenar metadados de ponto de restauração de aplicativos e para mover entre provedores de infraestrutura, e utlizado também para dados em massa. 
Da mesma forma, os perfis de localização também são usados para importar aplicativos para um cluster diferente do cluster de origem em que o aplicativo foi capturado.

# Upgrade de versão K10 Kasten
```sh
helm repo update && \
  helm get values k10 --output yaml --namespace=kasten-io > k10_val.yaml && \
  helm upgrade k10 kasten/k10 --namespace=kasten-io -f k10_val.yaml \
  --set injectKanisterSidecar.enabled=true \
  --set-string injectKanisterSidecar.objectSelector.matchLabels.component=db \
  --set-string injectKanisterSidecar.namespaceSelector.matchLabels.k10/injectKanisterSidecar=true
```
# Notas:
No caso de local de armazenamento de arquivo NFS, o compartilhamento NFS exportado deve ser acessível a partir do cluster de destino e montado em todos os nós onde o K10 está instalado.

# Varias leituras de referencias:

https://veducate.co.uk/kasten-multi-cluster/

https://horstmann.in/how-i-built-my-kubernetes-homelab-part-6/

https://www.luizpessol.com.br/2021/07/28/backup-kubernetes-com-kasten-k10/

https://www.infracloud.io/blogs/k8s-disaster-recovery-using-kasten-k10/

https://horstmann.in/how-i-built-my-kubernetes-homelab-part-6/

https://www.unixarena.com/2021/09/kubernetes-backup-kasten-k10-test-drive.html/

https://veducate.co.uk/kasten-tanzu/
