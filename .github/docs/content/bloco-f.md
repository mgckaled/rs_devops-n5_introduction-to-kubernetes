<!-- markdownlint-disable -->
# Bloco F - Entendendo Volumes no Kubernetes

## Resumo Executivo

A gestão de armazenamento persistente constitui desafio fundamental em ambientes de containers, onde a natureza efêmera dos Pods contrasta com requisitos de aplicações stateful que necessitam preservar dados além do ciclo de vida de containers individuais. O Kubernetes aborda esta complexidade através de sistema sofisticado de abstrações de armazenamento que desacopla consumidores de armazenamento (aplicações) dos provedores de armazenamento (infraestrutura física), permitindo portabilidade, flexibilidade e gestão declarativa de recursos de persistência.

Este documento explora sistematicamente os componentes fundamentais do subsistema de armazenamento persistente do Kubernetes: StorageClasses que abstraem características de diferentes backends de armazenamento, PersistentVolumes (PVs) que representam recursos físicos de armazenamento provisionados no cluster, PersistentVolumeClaims (PVCs) que expressam requisições de armazenamento por parte de usuários, e a integração destes componentes com Pods que efetivamente consomem o armazenamento. Esta arquitetura em camadas proporciona separação clara de responsabilidades entre administradores de cluster que provisionam capacidade de armazenamento e desenvolvedores que simplesmente requisitam armazenamento com características específicas.

A distinção entre volumes efêmeros e persistentes estabelece fundação conceitual crítica. Volumes efêmeros, como emptyDir ou configMap, existem apenas durante vida útil de Pods e são apropriados para dados temporários como caches ou diretórios de trabalho. Volumes persistentes, gerenciados através do subsistema PV/PVC, transcendem ciclos de vida de Pods individuais, preservando dados através de reinicializações, migrações e até mesmo destruição completa de Pods, tornando-os essenciais para aplicações como bancos de dados, sistemas de arquivos distribuídos e qualquer carga de trabalho que requeira durabilidade de dados.

O provisionamento de volumes pode ocorrer de duas formas distintas: estático, onde administradores pré-criam PersistentVolumes que usuários posteriormente reivindicam através de PVCs, ou dinâmico, onde PersistentVolumes são automaticamente criados em resposta a PVCs quando nenhum PV existente satisfaz os requisitos. O provisionamento dinâmico, facilitado por StorageClasses que encapsulam provisioners específicos de fornecedores, representa abordagem preferencial em ambientes modernos, eliminando necessidade de pré-provisionamento manual e permitindo alocação just-in-time de recursos de armazenamento. Este documento detalha mecânicas, configurações, melhores práticas e padrões de uso para todos estes componentes, formando guia abrangente para gestão de armazenamento persistente em Kubernetes.

## 1. Introdução e Conceitos

### 1.1. O Desafio do Armazenamento em Ambientes Containerizados

Containers fundamentam-se em princípio de imutabilidade e efemeridade, onde instâncias são criadas rapidamente de imagens imutáveis e destruídas quando não mais necessárias. Esta arquitetura oferece vantagens substanciais para escalabilidade, portabilidade e operações, mas introduz desafio fundamental para aplicações que necessitam persistir dados além do ciclo de vida de containers individuais.

Por padrão, sistemas de arquivos de containers são efêmeros - quando um container termina, todas as mudanças feitas ao sistema de arquivos são perdidas. Para aplicações stateless como servidores web que apenas processam requisições sem manter estado local, esta característica é aceitável ou até desejável. Entretanto, aplicações stateful como bancos de dados, sistemas de mensageria, ou aplicações que processam uploads de usuários requerem mecanismos que garantam persistência de dados independentemente do estado dos containers que os manipulam.

### 1.2. Evolução do Armazenamento no Kubernetes

O Kubernetes evoluiu significativamente suas capacidades de armazenamento ao longo das versões. Inicialmente, volumes eram definidos diretamente nas especificações de Pods, acoplando fortemente aplicações a detalhes de implementação de armazenamento específicos. Esta abordagem violava princípios de abstração e dificultava portabilidade de aplicações entre diferentes clusters ou provedores de infraestrutura.

A introdução do subsistema de PersistentVolume/PersistentVolumeClaim representou evolução fundamental, estabelecendo camada de indireção que permite desenvolvedores especificarem requisitos de armazenamento (tamanho, modos de acesso, características de performance) sem conhecimento ou dependência de detalhes de implementação subjacentes. Esta separação de preocupações alinha-se com princípios de infraestrutura como código e facilita gestão de aplicações em ambientes híbridos ou multi-cloud.

### 1.3. Volumes Efêmeros versus Persistentes

O Kubernetes suporta dois paradigmas fundamentalmente diferentes de armazenamento, cada um apropriado para diferentes casos de uso:

#### 1.3.1. Volumes Efêmeros

Volumes efêmeros existem apenas durante vida útil do Pod que os contém. Quando o Pod é deletado, o volume e todos os dados nele contidos são perdidos. Tipos comuns incluem:

- **emptyDir**: Diretório vazio criado quando Pod é atribuído a nó, existindo enquanto Pod executa naquele nó
- **configMap**: Fornece configurações através de volumes
- **secret**: Fornece informações sensíveis através de volumes
- **downwardAPI**: Expõe metadata do Pod como arquivos

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-volume-example
spec:
  containers:
  - name: application
    image: nginx
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: cache-volume
    emptyDir: {}
  - name: config-volume
    configMap:
      name: application-config
```

#### 1.3.2. Volumes Persistentes

Volumes persistentes transcendem ciclo de vida de Pods individuais, preservando dados através de reinicializações, reescalonamentos e migrações. São gerenciados através de PersistentVolumes e PersistentVolumeClaims, proporcionando durabilidade e disponibilidade de dados críticos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: persistent-volume-example
spec:
  containers:
  - name: database
    image: postgres:14
    volumeMounts:
    - name: data-volume
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: database-pvc
```

## 2. StorageClass: Abstraindo Backends de Armazenamento

### 2.1. Conceito e Propósito

StorageClass fornece mecanismo para administradores descreverem "classes" de armazenamento que oferecem. Diferentes classes podem mapear para níveis de qualidade de serviço, políticas de backup, ou políticas arbitrárias determinadas por administradores. O Kubernetes em si é agnóstico sobre o que classes representam - esta é decisão organizacional baseada em requisitos específicos.

Cada StorageClass contém campos que descrevem volumes pertencentes àquela classe: provisioner (que implementação de armazenamento será usada), parameters (parâmetros específicos do provisioner), e reclaimPolicy (o que acontece com volume quando claim é deletado).

### 2.2. Componentes de um StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
mountOptions:
  - debug
  - discard
```

#### 2.2.1. Provisioner

O provisioner determina qual plugin de volume será usado para provisionar PersistentVolumes. Kubernetes suporta provisioners internos para provedores cloud principais e externos através da interface CSI (Container Storage Interface).

Provisioners comuns incluem:
- **kubernetes.io/aws-ebs**: Amazon Elastic Block Store
- **kubernetes.io/azure-disk**: Azure Disk
- **kubernetes.io/gce-pd**: Google Compute Engine Persistent Disk
- **kubernetes.io/cinder**: OpenStack Cinder
- **kubernetes.io/no-provisioner**: Para volumes que requerem provisionamento manual

```yaml
# StorageClass para AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
```

#### 2.2.2. Parameters

Parameters são específicos do provisioner e passados diretamente para implementação de armazenamento subjacente. Cada provisioner define seu próprio conjunto de parâmetros suportados.

```yaml
# Exemplo: StorageClass GCE com parâmetros específicos
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gce-pd-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
  fsType: ext4
```

#### 2.2.3. Reclaim Policy

A política de recuperação determina o que acontece com PersistentVolume quando PersistentVolumeClaim que o utiliza é deletado. Duas políticas são suportadas:

- **Delete**: O PV e seus dados são deletados automaticamente quando PVC é deletado
- **Retain**: O PV permanece após PVC ser deletado, permitindo recuperação manual de dados

```yaml
reclaimPolicy: Retain  # Preservar dados após deleção de PVC
```

Para ambientes de produção com dados críticos, Retain é geralmente recomendado para prevenir perda acidental de dados.

#### 2.2.4. Volume Binding Mode

O modo de binding controla quando binding de volume e provisionamento dinâmico devem ocorrer:

- **Immediate** (padrão): Volume é bound e provisionado assim que PVC é criado
- **WaitForFirstConsumer**: Volume não é bound até que Pod usando PVC seja criado, permitindo scheduler considerar topologia

```yaml
volumeBindingMode: WaitForFirstConsumer
```

WaitForFirstConsumer é especialmente importante para volumes que têm restrições de topologia (como local volumes) ou quando scheduler precisa considerar localização de dados ao posicionar Pods.

#### 2.2.5. Allow Volume Expansion

Permite ou proíbe expansão de volumes após criação inicial.

```yaml
allowVolumeExpansion: true
```

Quando habilitado, usuários podem aumentar tamanho de PVCs editando o spec, e Kubernetes automaticamente expande volume subjacente.

### 2.3. StorageClass Padrão

Clusters podem ter StorageClass marcado como padrão. PVCs que não especificam storageClassName utilizarão automaticamente o StorageClass padrão.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

```bash
# Verificar StorageClass padrão
kubectl get storageclass
kubectl get sc

# Marcar StorageClass como padrão
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remover marca de StorageClass padrão
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### 2.4. Criação de StorageClass

```bash
# Criar StorageClass a partir de manifesto
kubectl apply -f storageclass.yaml

# Listar StorageClasses
kubectl get storageclass
kubectl get sc

# Descrever StorageClass
kubectl describe storageclass fast-ssd

# Deletar StorageClass
kubectl delete storageclass fast-ssd
```

### 2.5. Exemplos de StorageClass por Provedor

#### 2.5.1. Amazon Web Services (AWS)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

#### 2.5.2. Google Cloud Platform (GCP)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

#### 2.5.3. Microsoft Azure

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingMode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

#### 2.5.4. Ambientes Locais (kind/minikube)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

## 3. PersistentVolume: Representando Recursos de Armazenamento

### 3.1. Conceito e Características

PersistentVolume (PV) é recurso no cluster que representa pedaço de armazenamento provisionado por administrador ou dinamicamente provisionado usando StorageClasses. É recurso do cluster assim como nó é recurso do cluster. PVs são plugins de volume com ciclo de vida independente de qualquer Pod individual que utiliza o PV.

### 3.2. Ciclo de Vida de PersistentVolumes

PersistentVolumes progridem através de estados distintos durante seu ciclo de vida:

- **Available**: Recurso livre que ainda não foi bound a claim
- **Bound**: Volume está bound a claim
- **Released**: Claim foi deletado mas recurso ainda não foi reclamado pelo cluster
- **Failed**: Volume falhou em sua recuperação automática

```bash
# Ver status de PersistentVolumes
kubectl get pv
kubectl get persistentvolume

# Ver detalhes de PV específico
kubectl describe pv <pv-name>
```

### 3.3. Estrutura de um PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  labels:
    type: local
    app: database
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: "/mnt/data/postgres"
    type: DirectoryOrCreate
```

### 3.4. Capacity

Define capacidade de armazenamento do volume. Kubernetes suporta notação de quantidade de recursos padrão.

```yaml
capacity:
  storage: 10Gi  # 10 Gibibytes
```

Unidades suportadas:
- **E, P, T, G, M, K**: Potências de 1000 (decimal)
- **Ei, Pi, Ti, Gi, Mi, Ki**: Potências de 1024 (binário)

### 3.5. Access Modes

Modos de acesso definem como volume pode ser montado:

- **ReadWriteOnce (RWO)**: Volume pode ser montado como read-write por single nó
- **ReadOnlyMany (ROX)**: Volume pode ser montado como read-only por muitos nós
- **ReadWriteMany (RWX)**: Volume pode ser montado como read-write por muitos nós
- **ReadWriteOncePod (RWOP)**: Volume pode ser montado como read-write por single Pod (Kubernetes 1.22+)

```yaml
accessModes:
  - ReadWriteOnce
```

Importante notar que nem todos os backends de armazenamento suportam todos os modos. Por exemplo, EBS da AWS suporta apenas ReadWriteOnce, enquanto NFS suporta todos os três primeiros modos.

### 3.6. Persistent Volume Reclaim Policy

Determina o que acontece com PV quando PVC é deletado:

```yaml
persistentVolumeReclaimPolicy: Retain
```

- **Retain**: PV permanece com dados intactos, requerendo limpeza manual
- **Delete**: PV e dados são deletados automaticamente
- **Recycle** (deprecated): Executa rm -rf básico no volume

### 3.7. Tipos de Volumes

#### 3.7.1. HostPath

Monta arquivo ou diretório do filesystem do nó no Pod. Útil para desenvolvimento mas inadequado para produção multi-nó.

```yaml
hostPath:
  path: "/mnt/data"
  type: DirectoryOrCreate
```

Tipos de HostPath:
- **DirectoryOrCreate**: Cria diretório se não existir
- **Directory**: Diretório deve existir
- **FileOrCreate**: Cria arquivo se não existir
- **File**: Arquivo deve existir

#### 3.7.2. NFS

Monta share NFS existente no Pod.

```yaml
nfs:
  server: nfs-server.example.com
  path: "/exports/data"
```

#### 3.7.3. Cloud Provider Specific

```yaml
# AWS EBS
awsElasticBlockStore:
  volumeID: vol-0123456789abcdef0
  fsType: ext4

# GCE Persistent Disk
gcePersistentDisk:
  pdName: my-data-disk
  fsType: ext4

# Azure Disk
azureDisk:
  diskName: my-disk
  diskURI: /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Compute/disks/my-disk
```

### 3.8. Provisionamento Estático de PVs

```bash
# Criar PV
kubectl apply -f pv.yaml

# Verificar PV criado
kubectl get pv

# Ver detalhes
kubectl describe pv postgres-pv

# Deletar PV
kubectl delete pv postgres-pv
```

## 4. PersistentVolumeClaim: Requisitando Armazenamento

### 4.1. Conceito e Propósito

PersistentVolumeClaim (PVC) é requisição de armazenamento por usuário. É similar a Pod - Pods consomem recursos de nó e PVCs consomem recursos de PV. Pods podem requisitar níveis específicos de recursos (CPU e Memória). Claims podem requisitar tamanhos e modos de acesso específicos.

### 4.2. Estrutura de um PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  labels:
    app: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-storage
  selector:
    matchLabels:
      type: local
      app: database
```

### 4.3. Binding de PVC a PV

Quando PVC é criado, Kubernetes procura PV que satisfaça requisitos do claim. Se encontrado (e disponível), bind é estabelecido.

Processo de binding considera:
- **StorageClass**: Deve corresponder (ou ambos vazios)
- **Capacity**: PV deve ter capacidade >= requisitada
- **Access Modes**: PV deve suportar modos requisitados
- **Selector**: Se especificado, PV deve corresponder

```bash
# Criar PVC
kubectl apply -f pvc.yaml

# Ver status de PVC
kubectl get pvc

# Descrever PVC
kubectl describe pvc postgres-pvc

# Ver binding
kubectl get pvc postgres-pvc -o yaml | grep volumeName
```

### 4.4. Provisionamento Dinâmico

Quando nenhum PV estático satisfaz PVC, cluster pode tentar provisionar volume dinamicamente para PVC. Este provisionamento baseia-se em StorageClasses.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-ssd
```

Quando este PVC é criado, Kubernetes automaticamente:
1. Identifica StorageClass "fast-ssd"
2. Invoca provisioner especificado no StorageClass
3. Cria PV conforme parâmetros do StorageClass
4. Bind PVC ao novo PV

```bash
# Criar PVC com provisionamento dinâmico
kubectl apply -f dynamic-pvc.yaml

# Observar PV sendo criado automaticamente
kubectl get pv -w

# Ver PV dinamicamente criado
kubectl get pv
```

### 4.5. Selectors

Selectors permitem filtrar PVs disponíveis baseado em labels.

```yaml
selector:
  matchLabels:
    environment: production
    tier: database
  matchExpressions:
  - key: type
    operator: In
    values:
    - ssd
    - nvme
```

### 4.6. Volume Modes

Define se volume deve ser usado como filesystem ou block device.

```yaml
volumeMode: Filesystem  # Padrão
# ou
volumeMode: Block
```

### 4.7. Expandindo PVCs

Se StorageClass permite expansão (`allowVolumeExpansion: true`), PVCs podem ser expandidos após criação.

```bash
# Editar PVC para aumentar tamanho
kubectl edit pvc postgres-pvc
# Alterar spec.resources.requests.storage de 8Gi para 16Gi

# Verificar expansão
kubectl get pvc postgres-pvc
kubectl describe pvc postgres-pvc
```

## 5. Integrando Volumes com Aplicações

### 5.1. Montando PVC em Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      value: "secretpassword"
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
      subPath: postgres  # Importante para PostgreSQL
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

### 5.2. SubPath

O campo `subPath` permite montar subdiretório específico do volume ao invés de raiz.

```yaml
volumeMounts:
- name: shared-storage
  mountPath: /app/logs
  subPath: logs/application-a
- name: shared-storage
  mountPath: /app/cache
  subPath: cache/application-a
```

### 5.3. ReadOnly Mounts

Volumes podem ser montados como read-only.

```yaml
volumeMounts:
- name: config-storage
  mountPath: /etc/config
  readOnly: true
```

### 5.4. Montando em Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        volumeMounts:
        - name: html-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-storage
        persistentVolumeClaim:
          claimName: web-content-pvc
```

### 5.5. StatefulSets e Volume Claim Templates

StatefulSets permitem criar PVCs automaticamente para cada réplica.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database-cluster
spec:
  serviceName: "database"
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

Este manifesto cria automaticamente PVCs `data-database-cluster-0`, `data-database-cluster-1`, `data-database-cluster-2`.

## 6. Casos de Uso e Padrões

### 6.1. Bancos de Dados

```yaml
# PostgreSQL com volume persistente
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

### 6.2. Upload de Arquivos

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
spec:
  accessModes:
    - ReadWriteMany  # Permitir múltiplos Pods acessarem
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs-client

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: file-server
  template:
    metadata:
      labels:
        app: file-server
    spec:
      containers:
      - name: server
        image: nginx
        volumeMounts:
        - name: uploads
          mountPath: /usr/share/nginx/html/uploads
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: uploads-pvc
```

### 6.3. Logs Compartilhados

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: logs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-client

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: application
  template:
    metadata:
      labels:
        app: application
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
      - name: log-shipper
        image: fluent/fluentd:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
      volumes:
      - name: logs
        persistentVolumeClaim:
          claimName: logs-pvc
```

## 7. Melhores Práticas

### 7.1. Sempre Especificar StorageClass

```yaml
# BOM - Explícito
spec:
  storageClassName: fast-ssd

# EVITAR - Dependente de padrões do cluster
spec:
  storageClassName: ""  # Usa padrão ou falha
```

### 7.2. Usar Provisionamento Dinâmico

Preferir provisionamento dinâmico sobre estático quando possível, reduzindo overhead operacional e permitindo autoserviço.

### 7.3. Definir Resource Requests Apropriados

```yaml
resources:
  requests:
    storage: 10Gi  # Específico, não genérico
```

### 7.4. Escolher Access Mode Apropriado

```yaml
# Para banco de dados
accessModes:
  - ReadWriteOnce

# Para conteúdo compartilhado
accessModes:
  - ReadWriteMany
```

### 7.5. Usar Retain para Dados Críticos

```yaml
reclaimPolicy: Retain  # Em StorageClass ou PV
```

### 7.6. Implementar Backups

Volumes persistentes não são backups. Implementar estratégia de backup separada usando ferramentas como Velero, snapshots do provedor, ou scripts customizados.

### 7.7. Monitorar Uso de Armazenamento

```bash
# Ver uso de PVCs
kubectl get pvc --all-namespaces

# Monitorar capacidade
kubectl describe pvc <pvc-name>

# Ver PVs órfãos (Released)
kubectl get pv | grep Released
```

## 8. Troubleshooting

### 8.1. PVC Fica em Pending

```bash
# Verificar status e eventos
kubectl describe pvc <pvc-name>

# Verificar PVs disponíveis
kubectl get pv

# Verificar StorageClass
kubectl get storageclass <storage-class-name>
kubectl describe storageclass <storage-class-name>
```

Causas comuns:
- Nenhum PV disponível que satisfaça requisitos
- StorageClass não existe ou não configurado corretamente
- Provisioner não está funcionando
- Quotas de recursos excedidas

### 8.2. Pod Não Consegue Montar Volume

```bash
# Ver eventos do Pod
kubectl describe pod <pod-name>

# Ver logs do kubelet
kubectl logs -n kube-system kubelet-<node-name>

# Verificar se PVC está bound
kubectl get pvc <pvc-name>
```

Causas comuns:
- PVC não está bound
- Nó não consegue acessar backend de armazenamento
- Permissões de filesystem incorretas
- Volume já montado em outro nó (para RWO)

### 8.3. Volume Cheio

```bash
# Verificar uso dentro do Pod
kubectl exec <pod-name> -- df -h

# Expandir PVC (se suportado)
kubectl edit pvc <pvc-name>
# Aumentar spec.resources.requests.storage

# Verificar status de expansão
kubectl describe pvc <pvc-name>
```

### 8.4. PV em Estado Released

```bash
# Ver PVs released
kubectl get pv | grep Released

# Limpar manualmente e tornar disponível novamente
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'

# Ou deletar e recriar
kubectl delete pv <pv-name>
```

### 8.5. Comandos de Diagnóstico

```bash
# Listar todos os recursos de armazenamento
kubectl get storageclass,pv,pvc --all-namespaces

# Ver relacionamento entre PVC e PV
kubectl get pvc <pvc-name> -o yaml | grep volumeName

# Ver nó onde volume está montado
kubectl get pod <pod-name> -o wide

# Verificar eventos de armazenamento
kubectl get events --sort-by='.lastTimestamp' | grep -i volume

# Acessar diretório montado
kubectl exec -it <pod-name> -- ls -la /mnt/data
```

## 9. Conclusões

O subsistema de armazenamento persistente do Kubernetes representa uma das áreas mais críticas e complexas da plataforma, fornecendo abstrações sofisticadas que permitem aplicações stateful operarem efetivamente em ambientes de containers nativamente efêmeros. A arquitetura em camadas composta por StorageClasses, PersistentVolumes e PersistentVolumeClaims estabelece separação clara de responsabilidades que facilita gestão, portabilidade e operação de armazenamento em escala.

StorageClasses abstraem características e capacidades de diferentes backends de armazenamento, permitindo administradores definirem "classes de serviço" que encapsulam detalhes de implementação específicos de provedores. Esta abstração permite desenvolvedores requisitarem armazenamento baseado em características desejadas (performance, durabilidade, custo) sem necessidade de conhecimento profundo de tecnologias de armazenamento subjacentes ou diferenças entre provedores cloud. O suporte a provisionamento dinâmico elimina necessidade de pré-provisionamento manual de volumes, permitindo alocação just-in-time que otimiza utilização de recursos e reduz overhead operacional.

PersistentVolumes representam recursos físicos de armazenamento no cluster, enquanto PersistentVolumeClaims expressam requisições de armazenamento por usuários. Esta separação permite que administradores gerenciem capacidade de armazenamento independentemente de como ela é consumida, enquanto desenvolvedores podem requisitar armazenamento de forma self-service sem intervenção de operadores. O processo de binding automático entre PVCs e PVs, combinado com capacidades de provisionamento dinâmico, proporciona experiência fluida que equilibra autonomia de desenvolvedores com controles operacionais necessários.

A implementação efetiva de armazenamento persistente requer compreensão profunda de trade-offs entre diferentes modos de acesso, políticas de recuperação, e características de performance de backends de armazenamento. Access modes determinam como volumes podem ser compartilhados entre Pods e nós, reclaim policies controlam ciclo de vida de dados após deleção de claims, e escolhas de StorageClass impactam diretamente performance, disponibilidade e custo. StatefulSets complementam este ecossistema fornecendo primitivas especializadas para aplicações stateful que requerem identidades persistentes e armazenamento ordenado, tornando viável execução de bancos de dados distribuídos, sistemas de mensageria e outras cargas de trabalho complexas em Kubernetes.

## 10. Referências Bibliográficas

### 10.1. Documentação Oficial do Kubernetes

- Kubernetes Documentation. "Volumes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/volumes/

- Kubernetes Documentation. "Persistent Volumes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

- Kubernetes Documentation. "Storage Classes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/storage-classes/

- Kubernetes Documentation. "Dynamic Volume Provisioning". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/

### 10.2. Container Storage Interface (CSI)

- Kubernetes Documentation. "CSI Volume Plugins". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/volumes/#csi

- Container Storage Interface Specification. GitHub. Disponível em: https://github.com/container-storage-interface/spec

### 10.3. Provedores Cloud

- Amazon Web Services. "Amazon EBS CSI Driver". GitHub. Disponível em: https://github.com/kubernetes-sigs/aws-ebs-csi-driver

- Google Cloud. "Compute Engine Persistent Disk CSI Driver". GitHub. Disponível em: https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver

- Microsoft Azure. "Azure Disk CSI Driver". GitHub. Disponível em: https://github.com/kubernetes-sigs/azuredisk-csi-driver

### 10.4. Backup e Disaster Recovery

- Velero Documentation. "Velero - Backup and Migrate Kubernetes Resources". Velero.io. Disponível em: https://velero.io/docs/

- Kubernetes Documentation. "Volume Snapshots". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/storage/volume-snapshots/

## 11. Apêndices

### Apêndice A: Exemplo Completo de Stack de Aplicação

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: app-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain

---
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
  namespace: production
  labels:
    app: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: app-storage

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: production
type: Opaque
stringData:
  password: "changeMe123!"
  username: "admin"

---
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: appdb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: database-pvc

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

### Apêndice B: Scripts de Manutenção

```bash
#!/bin/bash
# volume-maintenance.sh

set -e

NAMESPACE="production"

# Função para listar PVCs e uso
list_pvc_usage() {
    echo "=== PVCs em $NAMESPACE ==="
    kubectl get pvc -n "$NAMESPACE"
    echo ""
    
    for pvc in $(kubectl get pvc -n "$NAMESPACE" -o jsonpath='{.items[*].metadata.name}'); do
        echo "PVC: $pvc"
        
        # Encontrar Pod usando PVC
        pod=$(kubectl get pods -n "$NAMESPACE" -o json | \
              jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$pvc\") | .metadata.name" | head -1)
        
        if [ -n "$pod" ]; then
            echo "  Pod: $pod"
            echo "  Uso de disco:"
            kubectl exec -n "$NAMESPACE" "$pod" -- df -h 2>/dev/null | grep -E "Filesystem|/var/lib" || echo "  Não foi possível verificar uso"
        else
            echo "  Nenhum Pod usando este PVC"
        fi
        echo ""
    done
}

# Função para limpar PVs Released
cleanup_released_pvs() {
    echo "=== PVs em estado Released ==="
    released_pvs=$(kubectl get pv | grep Released | awk '{print $1}')
    
    if [ -z "$released_pvs" ]; then
        echo "Nenhum PV Released encontrado"
        return
    fi
    
    for pv in $released_pvs; do
        echo "PV: $pv"
        read -p "Deletar este PV? (y/N): " confirm
        if [ "$confirm" = "y" ]; then
            kubectl delete pv "$pv"
            echo "PV $pv deletado"
        fi
    done
}

# Função para expandir PVC
expand_pvc() {
    local pvc_name=$1
    local new_size=$2
    
    if [ -z "$pvc_name" ] || [ -z "$new_size" ]; then
        echo "Uso: expand_pvc <pvc-name> <new-size>"
        return 1
    fi
    
    echo "Expandindo PVC $pvc_name para $new_size..."
    kubectl patch pvc "$pvc_name" -n "$NAMESPACE" -p "{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"$new_size\"}}}}"
    
    echo "Aguardando expansão..."
    kubectl wait --for=condition=FileSystemResizePending pvc/"$pvc_name" -n "$NAMESPACE" --timeout=60s || true
    
    # Reiniciar Pod para aplicar expansão
    pod=$(kubectl get pods -n "$NAMESPACE" -o json | \
          jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$pvc_name\") | .metadata.name" | head -1)
    
    if [ -n "$pod" ]; then
        echo "Reiniciando Pod $pod..."
        kubectl delete pod "$pod" -n "$NAMESPACE"
    fi
}

# Função para backup de dados de volume
backup_volume() {
    local pvc_name=$1
    local backup_path=$2
    
    if [ -z "$pvc_name" ] || [ -z "$backup_path" ]; then
        echo "Uso: backup_volume <pvc-name> <backup-path>"
        return 1
    fi
    
    echo "Criando backup de $pvc_name..."
    
    # Encontrar Pod usando PVC
    pod=$(kubectl get pods -n "$NAMESPACE" -o json | \
          jq -r ".items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName==\"$pvc_name\") | .metadata.name" | head -1)
    
    if [ -z "$pod" ]; then
        echo "Erro: Nenhum Pod encontrado usando PVC $pvc_name"
        return 1
    fi
    
    # Criar backup
    kubectl exec -n "$NAMESPACE" "$pod" -- tar czf - /var/lib/postgresql/data > "$backup_path/backup-$(date +%Y%m%d-%H%M%S).tar.gz"
    echo "Backup criado em $backup_path"
}

# Menu principal
case "${1:-help}" in
    list)
        list_pvc_usage
        ;;
    cleanup)
        cleanup_released_pvs
        ;;
    expand)
        expand_pvc "$2" "$3"
        ;;
    backup)
        backup_volume "$2" "$3"
        ;;
    *)
        echo "Uso: $0 {list|cleanup|expand|backup}"
        echo "  list                    - Listar PVCs e uso"
        echo "  cleanup                 - Limpar PVs Released"
        echo "  expand <pvc> <size>     - Expandir PVC"
        echo "  backup <pvc> <path>     - Backup de dados do volume"
        ;;
esac
```

### Apêndice C: Comandos Úteis

```bash
# Listar todos os recursos de armazenamento
kubectl get storageclass,pv,pvc --all-namespaces

# Ver detalhes de StorageClass
kubectl get storageclass -o yaml

# Criar StorageClass
kubectl apply -f storageclass.yaml

# Criar PVC
kubectl apply -f pvc.yaml

# Ver status de binding
kubectl get pvc -w

# Ver qual PV está bound a PVC
kubectl get pvc <pvc-name> -o jsonpath='{.spec.volumeName}'

# Ver eventos relacionados a volumes
kubectl get events --sort-by='.lastTimestamp' | grep -i volume

# Descrever PV
kubectl describe pv <pv-name>

# Descrever PVC
kubectl describe pvc <pvc-name>

# Ver pods usando um PVC específico
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="<pvc-name>") | .metadata.namespace + "/" + .metadata.name'

# Acessar shell em Pod e verificar volumes montados
kubectl exec -it <pod-name> -- /bin/bash
df -h

# Ver uso de espaço em volume
kubectl exec <pod-name> -- du -sh /var/lib/postgresql/data

# Copiar dados de volume para local
kubectl cp <pod-name>:/var/lib/postgresql/data ./backup/

# Copiar dados locais para volume
kubectl cp ./restore/ <pod-name>:/var/lib/postgresql/data/

# Expandir PVC
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Ver progresso de expansão
kubectl describe pvc <pvc-name> | grep -A 5 Conditions

# Limpar PV Released
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'

# Alterar reclaim policy de PV
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Ver PVs sem claims
kubectl get pv | grep Available

# Ver PVs órfãos
kubectl get pv | grep Released

# Deletar PVC (cuidado!)
kubectl delete pvc <pvc-name>

# Force delete PVC stuck
kubectl patch pvc <pvc-name> -p '{"metadata":{"finalizers":null}}'
kubectl delete pvc <pvc-name> --force --grace-period=0
```

### Apêndice D: Glossário e Termos Técnicos

**Access Mode**: Modo que define como volume pode ser montado (ReadWriteOnce, ReadOnlyMany, ReadWriteMany, ReadWriteOncePod).

**Binding**: Processo de associar PersistentVolumeClaim a PersistentVolume disponível que satisfaça requisitos.

**Capacity**: Quantidade de armazenamento disponível em PersistentVolume, expressa em unidades como Gi (Gibibytes) ou Ti (Tebibytes).

**Container Storage Interface (CSI)**: Interface padronizada que permite plugins de armazenamento de terceiros integrarem com Kubernetes.

**Dynamic Provisioning**: Criação automática de PersistentVolumes em resposta a PersistentVolumeClaims quando nenhum PV estático satisfaz requisitos.

**EmptyDir**: Tipo de volume efêmero criado quando Pod é atribuído a nó, existindo apenas durante vida do Pod.

**Ephemeral Volume**: Volume que existe apenas durante vida útil do Pod, sendo deletado quando Pod termina.

**HostPath**: Tipo de volume que monta arquivo ou diretório do filesystem do nó diretamente no Pod.

**Persistent Volume (PV)**: Recurso de armazenamento no cluster provisionado por administrador ou dinamicamente via StorageClass.

**Persistent Volume Claim (PVC)**: Requisição de armazenamento por usuário, especificando tamanho, modo de acesso e StorageClass desejados.

**Provisioner**: Componente que cria volumes dinamicamente, específico para cada tipo de backend de armazenamento.

**ReadWriteOnce (RWO)**: Modo de acesso onde volume pode ser montado como read-write por único nó.

**ReadOnlyMany (ROX)**: Modo de acesso onde volume pode ser montado como read-only por múltiplos nós.

**ReadWriteMany (RWX)**: Modo de acesso onde volume pode ser montado como read-write por múltiplos nós.

**ReadWriteOncePod (RWOP)**: Modo de acesso onde volume pode ser montado como read-write por único Pod no cluster.

**Reclaim Policy**: Política que determina o que acontece com PersistentVolume quando PersistentVolumeClaim é deletado (Retain, Delete, Recycle).

**Released**: Estado de PersistentVolume onde claim associado foi deletado mas recursos ainda não foram recuperados.

**Retain**: Política de recuperação onde PV e dados são preservados após deleção de PVC, requerendo limpeza manual.

**Static Provisioning**: Método onde administradores pré-criam PersistentVolumes que usuários posteriormente reivindicam através de PVCs.

**StorageClass**: Recurso que descreve "classe" de armazenamento, incluindo provisioner, parâmetros e políticas.

**SubPath**: Campo que permite montar subdiretório específico de volume ao invés de raiz inteira.

**Volume Binding Mode**: Controla quando binding de volume e provisionamento dinâmico ocorrem (Immediate ou WaitForFirstConsumer).

**VolumeClaimTemplate**: Template em StatefulSet que cria automaticamente PersistentVolumeClaim para cada réplica.

**Volume Expansion**: Capacidade de aumentar tamanho de PersistentVolumeClaim após criação inicial, se suportado por StorageClass.
