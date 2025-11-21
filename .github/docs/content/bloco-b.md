<!-- markdownlint-disable -->

# Kubernetes: Orquestrando Containers com Pods, ReplicaSets, Deployments e Services

## 1. Resumo Executivo

Este documento aborda os conceitos fundamentais e objetos essenciais do Kubernetes para orquestração de containers em produção. O foco está na compreensão teórica e prática de Pods, ReplicaSets, Deployments e Services, elementos que formam a base para execução, replicação, atualização e exposição de aplicações containerizadas. A transição da abordagem imperativa para a declarativa é explorada em profundidade, demonstrando como manifestos YAML permitem gerenciamento consistente, versionável e reproduzível de recursos. O documento também examina os desafios de disponibilidade, redundância e zero-downtime deployment, apresentando soluções arquiteturais que garantem resiliência e continuidade operacional em ambientes de produção.

## 2. Introdução e Conceitos

### 2.1. Hierarquia de Objetos no Kubernetes

O Kubernetes organiza recursos em uma hierarquia lógica que facilita o gerenciamento de aplicações containerizadas. Compreender essa hierarquia é fundamental para trabalhar efetivamente com a plataforma.

#### Níveis Hierárquicos

1. **Container**: Menor unidade de execução, encapsula a aplicação e suas dependências
2. **Pod**: Menor unidade de deployment no Kubernetes, agrupa um ou mais containers
3. **ReplicaSet**: Controlador que garante número específico de réplicas de Pods
4. **Deployment**: Controlador de nível superior que gerencia ReplicaSets e suas atualizações
5. **Service**: Abstração de rede que expõe Pods de forma consistente

### 2.2. Abordagens de Gerenciamento

O Kubernetes suporta duas abordagens principais para gerenciar recursos:

#### Imperativa

Na abordagem imperativa, comandos diretos são executados para criar, modificar ou deletar recursos:

```bash
# Criar pod imperativ amente
kubectl run nginx --image=nginx:1.14.2

# Escalar deployment
kubectl scale deployment nginx --replicas=5

# Atualizar imagem
kubectl set image deployment/nginx nginx=nginx:1.16.0
```

**Características**:

- Comandos diretos e imediatos
- Útil para operações pontuais e debugging
- Não mantém histórico de mudanças
- Dificulta rastreabilidade e reprodutibilidade
- Não recomendado para produção

#### Declarativa

Na abordagem declarativa, o estado desejado é especificado em arquivos de configuração (manifestos):

```bash
# Aplicar configuração declarativa
kubectl apply -f deployment.yaml

# Aplicar múltiplos arquivos
kubectl apply -f ./manifests/

# Aplicar com namespace específico
kubectl apply -f deployment.yaml -n production
```

**Características**:

- Estado desejado definido em arquivos YAML ou JSON
- Versionável via sistemas de controle de versão (Git)
- Reproduzível e auditável
- Fonte única da verdade (single source of truth)
- Recomendado para ambientes de produção
- Facilita práticas de GitOps

### 2.3. Namespaces: Organização e Isolamento

Namespaces fornecem isolamento lógico de recursos dentro de um cluster Kubernetes.

#### Propósito dos Namespaces

- **Organização**: Segregar recursos por ambiente, equipe ou aplicação
- **Isolamento**: Separar recursos logicamente dentro do cluster
- **Controle de Acesso**: Aplicar políticas RBAC específicas
- **Quotas de Recursos**: Limitar consumo de CPU e memória por namespace
- **Governança**: Facilitar gestão multi-tenant

#### Namespaces Padrão

```bash
# Listar namespaces
kubectl get namespaces

# Namespaces do sistema
# default: namespace padrão para recursos sem namespace especificado
# kube-system: componentes do sistema Kubernetes
# kube-public: recursos públicos acessíveis a todos
# kube-node-lease: objetos de lease para heartbeat de nodes
```

#### Criação e Uso

Imperativo:

```bash
# Criar namespace
kubectl create namespace development

# Listar pods em namespace específico
kubectl get pods -n development
```

Declarativo:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: backend
```

### 2.4. Gestão de Recursos Computacionais

O Kubernetes permite especificar requisitos e limites de recursos para garantir qualidade de serviço e evitar contenção de recursos.

#### Requests e Limits

**Requests**: Quantidade garantida de recursos

- Usada pelo Scheduler para decidir em qual Node alocar o Pod
- Garante que o recurso estará disponível para o container
- Se o Node não tiver recursos suficientes, o Pod não será agendado

**Limits**: Quantidade máxima de recursos

- Define o teto de consumo do container
- Se excedido, o container pode ser throttled (CPU) ou killed (memória)
- Protege o Node contra esgotamento de recursos

#### Exemplo de Configuração

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"      # 250 milicores = 0.25 CPU
  limits:
    memory: "128Mi"
    cpu: "500m"      # 500 milicores = 0.5 CPU
```

#### Unidades de Medida

**CPU**:
- Expressa em cores ou milicores (m)
- 1 core = 1000m
- Exemplos: 100m (0.1 core), 500m (0.5 core), 2 (2 cores)

**Memória**:
- Expressa em bytes com sufixos
- Ki, Mi, Gi (base 1024)
- K, M, G (base 1000)
- Exemplos: 128Mi, 1Gi, 500M

#### Consequências de Configuração

**Sem requests nem limits**:
- Pod pode ser agendado em qualquer Node
- Pode consumir todos os recursos disponíveis
- Sujeito a eviction se Node ficar sob pressão

**Somente requests**:
- Scheduler garante recursos mínimos
- Pode consumir mais se disponível (burst)
- Risco de afetar outros Pods

**Requests e limits iguais**:
- Classe QoS: Guaranteed
- Recursos reservados exclusivamente
- Menor probabilidade de eviction
- Recomendado para cargas críticas

**Requests menores que limits**:
- Classe QoS: Burstable
- Permite uso elástico de recursos
- Balanceia garantia e flexibilidade

## 3. Pods: Unidade Fundamental de Deployment

### 3.1. Conceito e Arquitetura

O Pod é a menor unidade computacional que pode ser criada e gerenciada no Kubernetes. Representa um ou mais containers executando juntos no mesmo Node, compartilhando recursos de rede e armazenamento.

#### Características Principais

**Efemeridade**:
- Pods são descartáveis e substituíveis
- Não devem manter estado local persistente
- Podem ser criados e destruídos a qualquer momento
- IP do Pod pode mudar ao ser recriado

**Colocação**:
- Todos os containers de um Pod executam no mesmo Node
- Compartilham o mesmo namespace de rede (localhost)
- Compartilham volumes definidos no Pod
- Scheduled como unidade atômica

**Padrões de Uso**:

1. **Um container por Pod** (mais comum):
   - Pod encapsula um único container
   - Container executa a aplicação principal
   - Simplicidade e clareza

2. **Múltiplos containers por Pod**:
   - Sidecar pattern: container auxiliar (logging, proxy)
   - Ambassador pattern: proxy para serviços externos
   - Adapter pattern: normalização de dados/logs

### 3.2. Ciclo de Vida do Pod

#### Fases do Pod

1. **Pending**: Pod aceito mas containers não criados
2. **Running**: Pod atribuído a Node e ao menos um container executando
3. **Succeeded**: Todos containers terminaram com sucesso
4. **Failed**: Todos containers terminaram e ao menos um falhou
5. **Unknown**: Estado do Pod não pode ser determinado

#### Condições do Pod

- **PodScheduled**: Pod foi agendado para um Node
- **ContainersReady**: Todos containers do Pod estão prontos
- **Initialized**: Todos init containers completaram com sucesso
- **Ready**: Pod pode servir requisições

### 3.3. Manifesto de Pod

#### Estrutura Básica

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: development
  labels:
    app: nginx
    tier: frontend
    version: "1.14"
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

#### Componentes do Manifesto

**apiVersion**: Versão da API do Kubernetes
- Pods usam `v1` (core API)

**kind**: Tipo do objeto Kubernetes
- Especifica que é um Pod

**metadata**: Informações sobre o objeto
- **name**: Identificador único no namespace
- **namespace**: Namespace onde o Pod será criado
- **labels**: Pares chave-valor para identificação e seleção

**spec**: Especificação do Pod
- **containers**: Lista de containers do Pod
  - **name**: Nome do container
  - **image**: Imagem Docker a ser utilizada
  - **ports**: Portas expostas pelo container
  - **resources**: Requisitos e limites de recursos

### 3.4. Versionamento de Imagens

O versionamento adequado de imagens é crítico para rastreabilidade e rollback.

#### Estratégias de Versionamento

**Tag específica** (recomendado):

```yaml
image: nginx:1.14.2
```

- Garante versão exata da imagem
- Deployment reproduzível
- Facilita rollback
- Previne mudanças inesperadas

**Tag latest** (não recomendado para produção):

```yaml
image: nginx:latest
```

- Versão não determinística
- Pode mudar sem controle
- Dificulta debugging
- Problemas de reprodutibilidade

**SHA256 digest** (máxima garantia):

```yaml
image: nginx@sha256:abc123...
```

- Referência imutável
- Garante bit-a-bit a mesma imagem
- Ideal para auditoria e segurança

#### Boas Práticas

1. **Sempre use tags específicas em produção**
2. **Associe tag ao commit do código**
   - Exemplo: `myapp:commit-abc123` ou `myapp:v1.2.3`
3. **Implemente semantic versioning**
   - MAJOR.MINOR.PATCH (1.2.3)
4. **Automatize build e tag via CI/CD**
5. **Mantenha registro de mudanças (changelog)**

### 3.5. Acessando Pods

#### Port Forward

Permite acesso temporário a um Pod para debugging:

```bash
# Encaminhar porta local para porta do Pod
kubectl port-forward pod/nginx-pod 8080:80

# Acesso via navegador ou curl
curl http://localhost:8080
```

**Características**:
- Conexão direta entre máquina local e Pod
- Temporário e não persistente
- Ideal para debugging e testes
- Não recomendado para acesso permanente

#### Exec em Pod

Executar comandos dentro do container:

```bash
# Shell interativo
kubectl exec -it nginx-pod -- /bin/bash

# Comando específico
kubectl exec nginx-pod -- ls -la /usr/share/nginx/html

# Com múltiplos containers
kubectl exec -it nginx-pod -c nginx -- /bin/bash
```

#### Logs do Pod

```bash
# Ver logs
kubectl logs nginx-pod

# Seguir logs em tempo real
kubectl logs -f nginx-pod

# Logs de container específico
kubectl logs nginx-pod -c nginx

# Últimas N linhas
kubectl logs --tail=100 nginx-pod
```

### 3.6. Limitações dos Pods Standalone

Pods criados diretamente apresentam problemas significativos:

1. **Sem auto-recuperação**: Pod deletado não é recriado
2. **Sem replicação**: Impossível ter múltiplas instâncias
3. **Sem balanceamento**: Tráfego não distribuído
4. **Sem rolling updates**: Atualizações causam downtime
5. **Gerenciamento manual**: Escalabilidade limitada

Esses problemas são resolvidos por controladores como ReplicaSets e Deployments.

## 4. ReplicaSets: Garantindo Redundância

### 4.1. Conceito e Propósito

ReplicaSet é um controlador que garante que um número especificado de réplicas de Pods esteja sempre em execução.

#### Função Principal

- Manter número desejado de Pods idênticos
- Substituir Pods que falham ou são deletados
- Escalar número de réplicas para cima ou para baixo
- Garantir redundância e disponibilidade

#### Padrão de Reconciliação

O ReplicaSet opera em loop contínuo:

1. **Observa** o estado atual (quantos Pods estão rodando)
2. **Compara** com o estado desejado (réplicas especificadas)
3. **Age** para convergir ao estado desejado
   - Cria Pods se houver menos que o desejado
   - Deleta Pods se houver mais que o desejado

### 4.2. Manifesto de ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  namespace: development
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

#### Componentes Críticos

**spec.replicas**:
- Número desejado de Pods
- ReplicaSet mantém essa quantidade constantemente

**spec.selector**:
- Define como ReplicaSet identifica Pods gerenciados
- Usa labels para seleção
- **matchLabels**: Igualdade exata de labels
- **matchExpressions**: Seleção baseada em expressões

**spec.template**:
- Template usado para criar novos Pods
- Contém mesma estrutura de Pod
- **metadata.labels**: Devem corresponder ao selector
- **spec**: Especificação completa do Pod

### 4.3. Labels e Selectors

Labels são pares chave-valor anexados a objetos Kubernetes. Selectors permitem identificar conjuntos de objetos.

#### Importância dos Labels

- Identificação e organização de recursos
- Seleção para controladores (ReplicaSet, Service)
- Queries e filtragem
- Rastreamento de propriedade e propósito

#### Correspondência Obrigatória

```yaml
# Labels no template devem corresponder ao selector
selector:
  matchLabels:
    app: nginx
    tier: frontend

template:
  metadata:
    labels:
      app: nginx        # Deve corresponder
      tier: frontend    # Deve corresponder
```

Se os labels não corresponderem, o ReplicaSet não conseguirá gerenciar os Pods criados.

### 4.4. Comportamento e Auto-Recuperação

#### Criação Automática

Quando ReplicaSet é criado:
- Verifica quantos Pods com labels correspondentes existem
- Cria Pods até atingir número de réplicas desejado
- Distribui Pods entre Nodes disponíveis (via Scheduler)

#### Recuperação de Falhas

Quando Pod é deletado ou falha:
- ReplicaSet detecta divergência do estado desejado
- Cria automaticamente novo Pod para substituir
- Novo Pod pode ser criado no mesmo ou diferente Node
- Processo é automático e contínuo

```bash
# Deletar pod gerenciado por ReplicaSet
kubectl delete pod nginx-replicaset-abc123

# ReplicaSet cria automaticamente novo pod
# Verificar
kubectl get pods -w  # Watch mode
```

### 4.5. Escalabilidade

#### Escala Declarativa

```yaml
spec:
  replicas: 5  # Aumentar de 3 para 5
```

```bash
kubectl apply -f replicaset.yaml
```

#### Escala Imperativa

```bash
# Escalar para 5 réplicas
kubectl scale replicaset nginx-replicaset --replicas=5

# Verificar
kubectl get replicaset nginx-replicaset
```

#### Escala Automática (HPA)

Horizontal Pod Autoscaler ajusta réplicas automaticamente baseado em métricas:

```bash
kubectl autoscale replicaset nginx-replicaset --min=3 --max=10 --cpu-percent=80
```

### 4.6. Limitações do ReplicaSet

Apesar de resolver problemas de redundância, ReplicaSets apresentam limitações críticas:

#### Problema de Atualização de Versão

Ao atualizar a imagem do container:

```yaml
# Versão antiga
template:
  spec:
    containers:
    - name: nginx
      image: nginx:1.14.2

# Tentativa de atualização
template:
  spec:
    containers:
    - name: nginx
      image: nginx:1.16.0  # Nova versão
```

**Comportamento**:
- ReplicaSet **não** atualiza Pods existentes
- Novos Pods criados usarão nova imagem
- Pods existentes continuam com imagem antiga
- Necessário deletar manualmente Pods antigos

#### Processo Manual de Atualização

1. Aplicar manifesto com nova imagem
2. Deletar ReplicaSet com `--cascade=orphan` (mantém Pods)
3. Criar novo ReplicaSet
4. Deletar Pods antigos gradualmente

**Problemas**:
- Processo manual e propenso a erros
- Causa downtime
- Sem controle de rollout
- Sem histórico de versões
- Sem rollback automático

Esses problemas são resolvidos pelo Deployment.

## 5. Deployments: Gerenciamento de Atualizações

### 5.1. Conceito e Arquitetura

Deployment é um controlador de nível superior que gerencia ReplicaSets e fornece capacidades declarativas de atualização.

#### Hierarquia

```text
Deployment
  └─> ReplicaSet (versão nova)
        └─> Pod
        └─> Pod
        └─> Pod
  └─> ReplicaSet (versão antiga) - escalado para 0
        └─> (sem pods)
```

#### Responsabilidades

1. **Gerenciar ReplicaSets**: Cria e controla ReplicaSets
2. **Rolling Updates**: Atualiza Pods gradualmente
3. **Rollback**: Reverte para versões anteriores
4. **Histórico de Revisões**: Mantém registro de mudanças
5. **Declarativo**: Estado desejado em manifesto

### 5.2. Manifesto de Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: development
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

#### Diferenças em Relação ao ReplicaSet

- **apiVersion**: `apps/v1`
- **kind**: `Deployment` ao invés de `ReplicaSet`
- **spec.strategy**: Define estratégia de atualização (novo campo)
- Demais campos idênticos ao ReplicaSet

### 5.3. Estratégias de Atualização

#### RollingUpdate (Padrão)

Atualização gradual que mantém disponibilidade:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # Pods extras permitidos durante update
    maxUnavailable: 1     # Pods indisponíveis tolerados
```

**maxSurge**:
- Número ou percentual de Pods extras durante update
- Valor absoluto (ex: 1, 2) ou percentual (ex: 25%, 50%)
- Permite criar Pods novos antes de deletar antigos
- Default: 25%

**maxUnavailable**:
- Número ou percentual de Pods que podem estar indisponíveis
- Valor absoluto ou percentual
- Controla quantos Pods antigos deletar por vez
- Default: 25%

**Exemplo de Rollout**:

Estado inicial: 3 réplicas rodando v1.14.2

```text
maxSurge=1, maxUnavailable=1, replicas=3

Passo 1: Cria 1 novo Pod (v1.16.0)
  [v1.14.2] [v1.14.2] [v1.14.2] [v1.16.0]  <- 4 pods (surge)

Passo 2: Deleta 1 Pod antigo
  [v1.14.2] [v1.14.2] [v1.16.0]  <- 3 pods

Passo 3: Cria 1 novo Pod
  [v1.14.2] [v1.14.2] [v1.16.0] [v1.16.0]  <- 4 pods

Passo 4: Deleta 1 Pod antigo
  [v1.14.2] [v1.16.0] [v1.16.0]  <- 3 pods

Passo 5: Cria 1 novo Pod
  [v1.14.2] [v1.16.0] [v1.16.0] [v1.16.0]  <- 4 pods

Passo 6: Deleta último Pod antigo
  [v1.16.0] [v1.16.0] [v1.16.0]  <- 3 pods (completo)
```

**Vantagens**:
- Zero downtime deployment
- Transição gradual e controlada
- Permite rollback durante processo
- Mantém disponibilidade

#### Recreate

Deleta todos Pods antigos antes de criar novos:

```yaml
strategy:
  type: Recreate
```

**Comportamento**:
1. Escala ReplicaSet antigo para 0 (deleta todos Pods)
2. Espera todos Pods terminarem
3. Cria ReplicaSet novo
4. Escala para número de réplicas desejado

**Características**:
- Causa downtime
- Todos Pods são da mesma versão simultaneamente
- Mais simples e rápido
- Útil quando aplicações não suportam múltiplas versões simultâneas

### 5.4. Processo de Atualização

#### Atualização Declarativa

1. Modificar manifesto (ex: mudar imagem)

```yaml
containers:
- name: nginx
  image: nginx:1.16.0  # Atualizado de 1.14.2
```

2. Aplicar mudança

```bash
kubectl apply -f deployment.yaml
```

3. Deployment automaticamente:
   - Cria novo ReplicaSet com nova versão
   - Escala novo ReplicaSet gradualmente
   - Escala ReplicaSet antigo para baixo
   - Mantém ReplicaSet antigo (com 0 réplicas) para rollback

#### Atualização Imperativa

```bash
# Atualizar imagem
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.0

# Editar deployment diretamente
kubectl edit deployment nginx-deployment
```

#### Monitoramento do Rollout

```bash
# Status do rollout
kubectl rollout status deployment/nginx-deployment

# Histórico de revisões
kubectl rollout history deployment/nginx-deployment

# Detalhes de revisão específica
kubectl rollout history deployment/nginx-deployment --revision=2
```

### 5.5. Rollback

Reverter para versão anterior:

```bash
# Rollback para revisão anterior
kubectl rollout undo deployment/nginx-deployment

# Rollback para revisão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pausar rollout em andamento
kubectl rollout pause deployment/nginx-deployment

# Retomar rollout pausado
kubectl rollout resume deployment/nginx-deployment
```

#### Histórico de Revisões

```bash
kubectl rollout history deployment/nginx-deployment
```

Output:
```text
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply -f deployment.yaml
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.16.0
```

**revisionHistoryLimit**:
- Número de ReplicaSets antigos mantidos
- Default: 10
- Configurável em `spec.revisionHistoryLimit`

### 5.6. Controle de Disponibilidade

#### minReadySeconds

Tempo mínimo que Pod deve estar Ready antes de ser considerado disponível:

```yaml
spec:
  minReadySeconds: 30
```

- Previne rollout muito rápido
- Permite detecção de problemas em startup
- Útil com health checks

#### progressDeadlineSeconds

Tempo máximo para progresso do rollout:

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutos
```

- Se rollout não progride nesse tempo, marca como failed
- Permite detecção de rollouts travados
- Default: 600 segundos

## 6. Services: Expondo Aplicações

### 6.1. Conceito e Necessidade

Services fornecem abstração de rede estável para acessar Pods dinâmicos.

#### Problema Resolvido

Pods são efêmeros:
- IPs mudam quando Pods são recriados
- Número de Pods varia com escalabilidade
- Difícil saber qual Pod acessar
- Necessário balanceamento de carga

Service resolve:
- IP e DNS estáveis
- Descoberta automática de Pods
- Balanceamento de carga
- Abstração de endpoints dinâmicos

### 6.2. Funcionamento

Service usa labels para selecionar Pods:

```yaml
# Service
selector:
  app: nginx

# Pods matching
labels:
  app: nginx
```

Kubernetes automaticamente:
1. Identifica Pods com labels correspondentes
2. Adiciona IPs dos Pods como endpoints do Service
3. Atualiza endpoints quando Pods são criados/deletados
4. Distribui tráfego entre endpoints saudáveis

### 6.3. Tipos de Services

#### ClusterIP (Padrão)

Expõe Service em IP interno do cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: development
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80          # Porta do Service
    targetPort: 80    # Porta do container
```

**Características**:
- Acessível apenas dentro do cluster
- IP virtual estável (ClusterIP)
- DNS interno: `<service-name>.<namespace>.svc.cluster.local`
- Balanceamento de carga automático
- Uso: comunicação entre serviços internos

#### NodePort

Expõe Service em porta de todos os Nodes:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080   # Porta nos Nodes (30000-32767)
```

**Características**:
- Acessível externamente via `<NodeIP>:<NodePort>`
- Porta alocada em todos Nodes
- Range: 30000-32767 (configurável)
- Kubernetes roteia tráfego para Pods
- Uso: desenvolvimento, testes, acesso externo simples

**Exemplo de acesso**:
```bash
curl http://192.168.1.10:30080  # IP de qualquer Node
```

#### LoadBalancer

Provisiona load balancer externo (requer provedor cloud):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Características**:
- Cria load balancer no provedor de nuvem (AWS ELB, GCP LB, Azure LB)
- IP externo público
- Distribui tráfego entre Nodes
- Nodes distribuem para Pods
- Uso: produção, exposição pública de serviços

**Exemplo em AWS**:
```bash
kubectl get service nginx-loadbalancer
```

Output:
```text
NAME                  TYPE           EXTERNAL-IP
nginx-loadbalancer    LoadBalancer   abc-123.us-east-1.elb.amazonaws.com
```

#### ExternalName

Mapeia Service para nome DNS externo:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: db.example.com
```

**Características**:
- Não tem selector
- Retorna CNAME para nome externo
- Útil para integração com serviços externos
- Facilita migração e abstração

### 6.4. Manifesto Completo de Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: development
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
    tier: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  sessionAffinity: None
```

#### Campos Importantes

**spec.selector**:
- Labels para selecionar Pods
- Service roteia tráfego apenas para Pods matching

**spec.ports**:
- **port**: Porta exposta pelo Service
- **targetPort**: Porta do container (pode ser nome ou número)
- **protocol**: TCP ou UDP
- **name**: Nome da porta (útil com múltiplas portas)

**spec.sessionAffinity**:
- `None` (default): Sem afinidade, balanceamento round-robin
- `ClientIP`: Requisições do mesmo cliente vão para mesmo Pod

### 6.5. Endpoints

Endpoints são objetos que mapeiam Service para Pods:

```bash
# Listar endpoints
kubectl get endpoints nginx-service

# Detalhes
kubectl describe endpoints nginx-service
```

Output:
```text
NAME            ENDPOINTS
nginx-service   10.244.1.5:80,10.244.2.3:80,10.244.3.7:80
```

Cada endpoint representa um Pod:
- IP do Pod
- Porta do container
- Atualizado automaticamente

### 6.6. Descoberta de Serviços

#### DNS

Kubernetes fornece DNS interno:

```bash
# Formato: <service>.<namespace>.svc.cluster.local
nginx-service.development.svc.cluster.local

# Dentro do mesmo namespace
nginx-service

# Resolução
nslookup nginx-service.development.svc.cluster.local
```

#### Variáveis de Ambiente

Kubernetes injeta variáveis em Pods:

```bash
NGINX_SERVICE_SERVICE_HOST=10.96.0.10
NGINX_SERVICE_SERVICE_PORT=80
```

**Limitação**: Apenas para Services criados antes do Pod

### 6.7. Relação Service-Deployment

#### Separação de Responsabilidades

- **Deployment**: Gerencia execução e atualizações de Pods
- **Service**: Expõe Pods com endpoint estável

#### Relação Um-para-Um Típica

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx    # Label usado pelo Service
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx        # Seleciona Pods do Deployment
  ports:
  - port: 80
    targetPort: 80
```

#### Labels Conectam Deployment e Service

- Deployment define labels nos Pods
- Service usa selector com mesmos labels
- Mudanças nos Pods são refletidas automaticamente no Service
- Service permanece estável durante atualizações

## 7. Ciclo Completo de Deployment

### 7.1. Fluxo de Criação

1. **Aplicar Deployment**:

```bash
kubectl apply -f deployment.yaml -n development
```

2. **Deployment Controller**:
   - Cria ReplicaSet com hash da versão
   - Define número de réplicas

3. **ReplicaSet Controller**:
   - Cria Pods conforme template
   - Adiciona labels e owner references

4. **Scheduler**:
   - Atribui Pods a Nodes
   - Considera recursos, afinidades, taints

5. **Kubelet**:
   - Detecta Pods atribuídos ao seu Node
   - Instrui Container Runtime
   - Puxa imagens
   - Inicia containers

6. **Aplicar Service**:

```bash
kubectl apply -f service.yaml -n development
```

7. **Service Controller**:
   - Cria Service com ClusterIP
   - Identifica Pods via selector
   - Cria Endpoints

8. **Kube-proxy**:
   - Configura regras iptables ou IPVS
   - Habilita balanceamento de carga

### 7.2. Fluxo de Atualização

1. **Modificar Deployment** (ex: nova imagem)

```yaml
containers:
- name: nginx
  image: nginx:1.16.0  # Mudança
```

2. **Aplicar mudança**:

```bash
kubectl apply -f deployment.yaml
```

3. **Deployment Controller**:
   - Detecta mudança no template
   - Cria novo ReplicaSet com novo hash
   - Inicia rollout strategy

4. **Rolling Update** (se maxSurge=1, maxUnavailable=1):
   - Escala novo ReplicaSet: 0 → 1
   - Espera Pod Ready
   - Escala ReplicaSet antigo: 3 → 2
   - Repete até completar

5. **Service**:
   - Endpoints atualizados automaticamente
   - Novos Pods adicionados
   - Pods antigos removidos
   - Sem interrupção de serviço

6. **Rollout completo**:
   - ReplicaSet novo: 3 réplicas
   - ReplicaSet antigo: 0 réplicas (mantido para rollback)

### 7.3. Verificação e Troubleshooting

```bash
# Status do Deployment
kubectl get deployment nginx-deployment

# Detalhes
kubectl describe deployment nginx-deployment

# ReplicaSets
kubectl get replicaset

# Pods
kubectl get pods -l app=nginx

# Service e Endpoints
kubectl get service nginx-service
kubectl get endpoints nginx-service

# Eventos
kubectl get events --sort-by='.lastTimestamp'

# Logs
kubectl logs -l app=nginx --tail=100

# Rollout status
kubectl rollout status deployment/nginx-deployment
```

## 8. Conclusões

### Principais Aprendizados

1. **Hierarquia de Objetos**:
   - Pods são unidades fundamentais mas efêmeras
   - ReplicaSets garantem redundância mas não gerenciam atualizações
   - Deployments fornecem gerenciamento completo de ciclo de vida
   - Services abstraem acesso a Pods dinâmicos

2. **Abordagem Declarativa**:
   - Manifestos YAML definem estado desejado
   - Versionamento via Git
   - Reprodutibilidade e auditoria
   - Fonte única da verdade

3. **Gestão de Recursos**:
   - Requests garantem recursos mínimos
   - Limits protegem contra consumo excessivo
   - QoS classes determinam prioridade de eviction
   - Essencial para estabilidade do cluster

4. **Disponibilidade e Resiliência**:
   - ReplicaSets garantem número de réplicas
   - Rolling updates permitem zero-downtime
   - Rollback automático em caso de falha
   - Self-healing automático

5. **Abstração de Rede**:
   - Services fornecem endpoints estáveis
   - Descoberta automática via DNS
   - Balanceamento de carga nativo
   - Separação de responsabilidades (compute vs network)

### Melhores Práticas

1. **Sempre use controladores** (Deployment/ReplicaSet), nunca Pods standalone
2. **Adote abordagem declarativa** com manifestos versionados
3. **Especifique requests e limits** para todos containers
4. **Use tags específicas de imagens**, não latest
5. **Configure health checks** (liveness, readiness probes)
6. **Organize com namespaces** por ambiente ou equipe
7. **Use labels consistentes** para identificação
8. **Mantenha histórico de rollout** com annotations
9. **Teste rollouts em staging** antes de produção
10. **Monitore métricas e logs** continuamente

### Próximos Passos

Os conceitos apresentados formam a base para tópicos avançados:

- **ConfigMaps e Secrets**: Gerenciamento de configuração
- **Volumes e Persistent Storage**: Dados persistentes
- **Health Checks**: Liveness e Readiness Probes
- **Ingress**: Exposição HTTP/HTTPS avançada
- **Network Policies**: Segurança de rede
- **RBAC**: Controle de acesso baseado em funções
- **Helm**: Gerenciamento de pacotes Kubernetes
- **Operators**: Automação customizada

### Recomendações Finais

Para equipes implementando Kubernetes:

1. **Comece simples**: Deploy básico com Deployment e Service
2. **Incremente gradualmente**: Adicione complexidade conforme necessário
3. **Automatize tudo**: CI/CD, testes, monitoramento
4. **Documente manifestos**: Comentários e READMEs explicativos
5. **Estabeleça governança**: Padrões, reviews, políticas
6. **Capacite equipe**: Treinamento contínuo é essencial

## 9. Referências Bibliográficas

### Documentação Oficial

- Kubernetes Documentation. "Pods". Disponível em: https://kubernetes.io/docs/concepts/workloads/pods/. Acesso em: 2024.

- Kubernetes Documentation. "ReplicaSet". Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/. Acesso em: 2024.

- Kubernetes Documentation. "Deployments". Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/. Acesso em: 2024.

- Kubernetes Documentation. "Service". Disponível em: https://kubernetes.io/docs/concepts/services-networking/service/. Acesso em: 2024.

- Kubernetes Documentation. "Namespaces". Disponível em: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/. Acesso em: 2024.

- Kubernetes Documentation. "Labels and Selectors". Disponível em: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/. Acesso em: 2024.

- Kubernetes Documentation. "Managing Resources". Disponível em: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/. Acesso em: 2024.

### API Reference

- Kubernetes API Reference. "Pod v1 core". Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/. Acesso em: 2024.

- Kubernetes API Reference. "ReplicaSet v1 apps". Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/replica-set-v1/. Acesso em: 2024.

- Kubernetes API Reference. "Deployment v1 apps". Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/. Acesso em: 2024.

- Kubernetes API Reference. "Service v1 core". Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/. Acesso em: 2024.

### Livros e Publicações

- Burns, Brendan; Beda, Joe; Hightower, Kelsey. "Kubernetes: Up and Running". O'Reilly Media, 2019.

- Luksa, Marko. "Kubernetes in Action". Manning Publications, 2018.

- Hightower, Kelsey; Burns, Brendan; Beda, Joe. "Kubernetes Best Practices". O'Reilly Media, 2019.

### Artigos Técnicos

- Kubernetes Blog. "Kubernetes Deployment Strategies". Disponível em: https://kubernetes.io/blog/. Acesso em: 2024.

## 10. Apêndice

### Apêndice A: Comandos kubectl para Workloads

#### Pods

```bash
# Listar pods
kubectl get pods
kubectl get pods -n development
kubectl get pods --all-namespaces
kubectl get pods -o wide

# Detalhes do pod
kubectl describe pod nginx-pod

# Logs
kubectl logs nginx-pod
kubectl logs -f nginx-pod
kubectl logs nginx-pod -c nginx
kubectl logs --tail=100 nginx-pod

# Exec
kubectl exec nginx-pod -- ls
kubectl exec -it nginx-pod -- /bin/bash

# Port forward
kubectl port-forward nginx-pod 8080:80

# Deletar
kubectl delete pod nginx-pod
```

#### ReplicaSets

```bash
# Listar
kubectl get replicaset
kubectl get rs

# Detalhes
kubectl describe replicaset nginx-replicaset

# Escalar
kubectl scale replicaset nginx-replicaset --replicas=5

# Deletar (deleta pods também)
kubectl delete replicaset nginx-replicaset

# Deletar (mantém pods)
kubectl delete replicaset nginx-replicaset --cascade=orphan
```

#### Deployments

```bash
# Listar
kubectl get deployment
kubectl get deploy

# Detalhes
kubectl describe deployment nginx-deployment

# Criar
kubectl apply -f deployment.yaml

# Atualizar imagem
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.0

# Escalar
kubectl scale deployment nginx-deployment --replicas=5

# Autoscale
kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80

# Rollout status
kubectl rollout status deployment/nginx-deployment

# Rollout history
kubectl rollout history deployment/nginx-deployment

# Rollback
kubectl rollout undo deployment/nginx-deployment

# Rollback para revisão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pausar rollout
kubectl rollout pause deployment/nginx-deployment

# Retomar rollout
kubectl rollout resume deployment/nginx-deployment

# Deletar
kubectl delete deployment nginx-deployment
```

#### Services

```bash
# Listar
kubectl get service
kubectl get svc

# Detalhes
kubectl describe service nginx-service

# Endpoints
kubectl get endpoints nginx-service

# Criar
kubectl apply -f service.yaml

# Expor deployment (imperativo)
kubectl expose deployment nginx-deployment --port=80 --type=ClusterIP

# Deletar
kubectl delete service nginx-service
```

### Apêndice B: Exemplos de Manifestos Completos

#### Deployment com Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
  labels:
    app: nginx
    version: "1.16"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "1.16"
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.0
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  revisionHistoryLimit: 10
```

#### Service com Múltiplas Portas

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

#### Namespace com ResourceQuota

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

### Apêndice C: Estratégias de Rolling Update

#### Conservadora (Alta Disponibilidade)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

- Sempre mantém todas réplicas disponíveis
- Rollout mais lento
- Requer recursos extras temporariamente
- Ideal para serviços críticos

#### Balanceada

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

- Valores default
- Balanceia velocidade e disponibilidade
- Adequado para maioria dos casos

#### Agressiva (Rápida)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 100%
    maxUnavailable: 50%
```

- Rollout rápido
- Aceita maior indisponibilidade
- Usa mais recursos temporariamente
- Adequado para ambientes de teste

### Apêndice D: Troubleshooting Comum

#### Pod não inicia (ImagePullBackOff)

```bash
# Verificar
kubectl describe pod nginx-pod

# Possíveis causas
# - Nome de imagem incorreto
# - Tag não existe
# - Registry requer autenticação
# - Sem acesso à internet

# Solução
# - Verificar nome e tag da imagem
# - Configurar ImagePullSecrets se necessário
```

#### Pod em CrashLoopBackOff

```bash
# Verificar logs
kubectl logs nginx-pod
kubectl logs nginx-pod --previous

# Possíveis causas
# - Aplicação falha ao iniciar
# - Comando incorreto
# - Falta variáveis de ambiente
# - Recursos insuficientes

# Solução
# - Verificar logs de erro
# - Validar configuração
# - Aumentar resources.limits se OOMKilled
```

#### Service não roteia tráfego

```bash
# Verificar endpoints
kubectl get endpoints nginx-service

# Se vazio
# - Labels do Service não correspondem aos Pods
# - Pods não estão Ready

# Verificar labels
kubectl get pods --show-labels
kubectl describe service nginx-service

# Solução
# - Corrigir selector do Service
# - Verificar readiness probes
```

#### Deployment travado

```bash
# Status
kubectl rollout status deployment/nginx-deployment

# Histórico
kubectl rollout history deployment/nginx-deployment

# Eventos
kubectl describe deployment nginx-deployment

# Possíveis causas
# - Nova imagem com problemas
# - Recursos insuficientes no cluster
# - Readiness probe falhando

# Solução
# - Rollback: kubectl rollout undo deployment/nginx-deployment
# - Verificar logs dos novos pods
# - Verificar eventos do cluster
```

### Apêndice E: Glossário e Termos Técnicos

**ClusterIP**: Tipo de Service que expõe aplicação apenas dentro do cluster com IP virtual interno.

**Declarativo**: Abordagem onde estado desejado é definido em arquivos, e o sistema converge automaticamente.

**Deployment**: Controlador que gerencia ReplicaSets e fornece atualizações declarativas de Pods.

**Downtime**: Período em que aplicação está indisponível para usuários.

**Endpoint**: IP e porta de um Pod que faz parte de um Service.

**Efemeridade**: Característica de recursos temporários e descartáveis que podem ser substituídos a qualquer momento.

**Health Check**: Verificação periódica de saúde de containers (liveness e readiness probes).

**Imperativo**: Abordagem onde comandos diretos especificam ações a executar.

**Label**: Par chave-valor anexado a objetos para identificação e seleção.

**Limits**: Quantidade máxima de recursos (CPU, memória) que container pode consumir.

**LoadBalancer**: Tipo de Service que provisiona load balancer externo em provedor de nuvem.

**maxSurge**: Número de Pods extras permitidos durante rolling update.

**maxUnavailable**: Número de Pods que podem estar indisponíveis durante rolling update.

**Manifesto**: Arquivo YAML ou JSON que define configuração declarativa de recursos Kubernetes.

**Namespace**: Isolamento lógico de recursos dentro de cluster Kubernetes.

**NodePort**: Tipo de Service que expõe aplicação em porta específica de todos Nodes.

**Pod**: Menor unidade de deployment no Kubernetes, agrupa um ou mais containers.

**Port Forward**: Encaminhamento de porta local para porta de Pod, usado para debugging.

**QoS (Quality of Service)**: Classificação de Pods (Guaranteed, Burstable, BestEffort) baseada em recursos.

**Reconciliação**: Processo de convergir estado atual ao estado desejado.

**Replica**: Cópia idêntica de Pod executando em cluster.

**ReplicaSet**: Controlador que mantém número estável de réplicas de Pods.

**Requests**: Quantidade garantida de recursos para container.

**Rollback**: Reversão para versão anterior de Deployment.

**Rolling Update**: Estratégia de atualização gradual mantendo disponibilidade.

**Rollout**: Processo de atualização de Deployment.

**Selector**: Critério baseado em labels para selecionar objetos.

**Service**: Abstração que expõe conjunto de Pods como serviço de rede.

**Strategy**: Estratégia de atualização de Deployment (RollingUpdate ou Recreate).

**Template**: Modelo usado por controladores para criar novos Pods.

**Zero-downtime deployment**: Atualização de aplicação sem interrupção de serviço.
