<!-- markdownlint-disable -->

# Bloco A - Kubernetes: Fundamentos de Orquestração de Containers

## Resumo Executivo

O Kubernetes representa uma das tecnologias mais transformadoras no campo da computação em nuvem moderna, estabelecendo-se como o orquestrador de containers de facto para aplicações distribuídas em escala. Originado do projeto Borg desenvolvido internamente pelo Google e posteriormente doado à Cloud Native Computing Foundation (CNCF), o Kubernetes automatiza processos complexos de implantação, escalabilidade e gerenciamento de aplicações containerizadas através de uma abordagem declarativa e extensível.

Este documento apresenta uma introdução abrangente aos conceitos fundamentais do Kubernetes, explorando desde os problemas que motivaram sua criação até os componentes arquiteturais que compõem um cluster funcional. A containerização de aplicações, embora resolva diversos desafios relacionados à portabilidade e consistência de ambientes, introduz complexidades operacionais significativas quando executada em escala. Questões como recuperação automática de falhas, distribuição de carga entre múltiplas instâncias, elasticidade dinâmica, controle granular de recursos computacionais e coordenação de aplicações multi-container demandam soluções sofisticadas que o Kubernetes provê de forma nativa.

A arquitetura do Kubernetes fundamenta-se na separação clara entre o plano de controle (control plane), responsável pela tomada de decisões e gerenciamento do estado do cluster, e os nós de trabalho (worker nodes), que executam as cargas de trabalho containerizadas. Esta separação permite escalabilidade horizontal, alta disponibilidade e isolamento de responsabilidades. Componentes como kube-apiserver, etcd, kube-scheduler, kube-controller-manager, kubelet e kube-proxy trabalham em conjunto para garantir que o estado desejado das aplicações, expresso através de manifestos declarativos, seja continuamente mantido no cluster.

A versatilidade do Kubernetes manifesta-se através de suas múltiplas opções de implantação, desde ambientes gerenciados oferecidos por provedores de nuvem pública até instalações auto-gerenciadas em infraestrutura on-premises. Para fins de aprendizado e desenvolvimento, ferramentas como kind, minikube e k3s permitem a execução de clusters locais completos, facilitando a experimentação e compreensão dos conceitos sem necessidade de infraestrutura complexa. A interação com clusters Kubernetes ocorre primariamente através do kubectl, interface de linha de comando que abstrai a complexidade da API REST subjacente, complementada por ferramentas visuais como Lens e k9s que proporcionam interfaces alternativas para operações comuns.

## 1. Introdução e Conceitos

### 1.1. Contexto Histórico e Evolução

A computação em nuvem moderna passou por múltiplas transformações significativas nas últimas duas décadas. Inicialmente, aplicações eram implantadas diretamente em servidores físicos dedicados, resultando em baixa utilização de recursos e custos operacionais elevados. A virtualização introduziu a capacidade de executar múltiplas máquinas virtuais em um único servidor físico, melhorando substancialmente a densidade de utilização de recursos. No entanto, máquinas virtuais carregam o overhead de sistemas operacionais completos, consumindo recursos significativos de memória e armazenamento.

A containerização emergiu como evolução natural deste paradigma, proporcionando isolamento de aplicações através de namespaces e cgroups do kernel Linux, sem o overhead de virtualização completa de hardware. Containers encapsulam aplicações e suas dependências em unidades portáteis e consistentes que executam de forma idêntica em qualquer ambiente que suporte a runtime de containers. Docker popularizou esta tecnologia através de ferramentas acessíveis e uma vasta biblioteca de imagens pré-construídas, democratizando a adoção de containers.

O Google acumulou mais de uma década de experiência executando bilhões de containers através de seus sistemas internos Borg e Omega. Em 2014, o Google doou o código que se tornou a base do Kubernetes à comunidade open source, aplicando as lições aprendidas na operação de sistemas distribuídos em escala planetária. A doação subsequente do projeto à Cloud Native Computing Foundation em 2015 estabeleceu governança neutra e acelerou a adoção empresarial através de contribuições de centenas de organizações.

### 1.2. Definição e Propósito do Kubernetes

Kubernetes, frequentemente abreviado como K8s (onde "8" representa as oito letras entre "K" e "s"), constitui uma plataforma open source para orquestração de containers que automatiza a implantação, escalabilidade e operação de aplicações containerizadas. O nome "Kubernetes" deriva do grego e significa "timoneiro" ou "piloto", refletindo metaforicamente seu papel de navegar e gerenciar aplicações através dos mares turbulentos da computação distribuída.

O propósito fundamental do Kubernetes transcende simplesmente executar containers. A plataforma fornece abstrações declarativas que permitem aos desenvolvedores especificar o estado desejado de suas aplicações, enquanto o sistema trabalha continuamente para reconciliar o estado atual com o desejado. Esta abordagem declarativa contrasta com modelos imperativos tradicionais, onde operadores devem especificar sequências exatas de comandos para alcançar um objetivo.

As capacidades centrais do Kubernetes incluem:

- **Descoberta de serviços e balanceamento de carga**: Distribuição automática de tráfego de rede entre instâncias de aplicações
- **Orquestração de armazenamento**: Montagem automática de sistemas de armazenamento locais, provedores de nuvem ou soluções de storage em rede
- **Rollouts e rollbacks automatizados**: Implantação gradual de mudanças com capacidade de reversão automática em caso de falhas
- **Empacotamento automático**: Alocação otimizada de containers em nós baseada em requisitos de recursos e restrições especificadas
- **Autocura**: Reinicialização automática de containers que falham, substituição de containers, eliminação de containers que não respondem a health checks
- **Gerenciamento de configurações e segredos**: Armazenamento e gestão de informações sensíveis sem reconstrução de imagens de containers

### 1.3. O Ecossistema Cloud Native e a CNCF

A Cloud Native Computing Foundation (CNCF) constitui uma organização subsidiária da Linux Foundation fundada em 2015 para promover e sustentar o ecossistema de tecnologias cloud-native. A missão da CNCF engloba tornar a computação cloud-native ubíqua através do fomento de projetos open source que definem o estado da arte em orquestração de containers, microsserviços, service mesh, observabilidade e segurança.

A CNCF serve como repositório neutro em relação a fornecedores para projetos críticos da infraestrutura tecnológica global, hospedando componentes essenciais incluindo Kubernetes, Prometheus, Envoy, etcd, e dezenas de outros projetos que formam a base de arquiteturas modernas de microsserviços. A fundação categoriza projetos em três níveis de maturidade:

- **Sandbox**: Projetos em estágio inicial que demonstram potencial mas requerem mais desenvolvimento e adoção
- **Incubating**: Projetos que alcançaram uso em ambientes de produção e demonstram crescimento de comunidade sustentável
- **Graduated**: Projetos que atingiram ampla adoção, maturidade técnica e governança robusta

Kubernetes foi anunciado em conjunto com a formação da CNCF em 2015, servindo como projeto semente da fundação, contribuído pelo Google. A governança neutra provida pela CNCF permitiu que competidores comerciais colaborassem no desenvolvimento do Kubernetes, resultando em inovação acelerada e padronização de interfaces. Atualmente, a CNCF conta com mais de 700 membros organizacionais e sustenta um ecossistema vibrante que abrange todo o ciclo de vida de aplicações cloud-native.

## 2. Cenário e Problemas Fundamentais

### 2.1. Desafios da Execução de Containers em Escala

A containerização de aplicações resolve problemas significativos relacionados à consistência de ambientes, densidade de recursos e velocidade de implantação. No entanto, a operação de containers em escala produtiva introduz complexidades operacionais substanciais que demandam soluções sistemáticas. Considere uma aplicação web moderna composta por múltiplos microsserviços, cada um executando em containers separados. A gestão manual desta infraestrutura rapidamente torna-se impraticável à medida que o número de serviços e instâncias cresce.

#### 2.1.1. Recuperação de Falhas e Resiliência

Containers individuais podem falhar por diversas razões: erros de aplicação, esgotamento de recursos, problemas de rede, ou falhas de hardware subjacente. Em ambientes de produção, a detecção rápida de falhas e recuperação automática constituem requisitos fundamentais para manter disponibilidade aceitável. Sem orquestração, cada falha requer intervenção manual para diagnosticar o problema, identificar o container afetado, e reiniciá-lo ou substituí-lo.

O Kubernetes implementa mecanismos sofisticados de health checking através de liveness probes e readiness probes. Liveness probes determinam se um container está executando corretamente e deve ser reiniciado se falhar. Readiness probes determinam se um container está pronto para aceitar tráfego. Esta separação permite que containers executem processos de inicialização complexos sem receber requisições prematuramente, enquanto mantém a capacidade de detectar e recuperar de estados de deadlock ou corrupção que ocorram durante operação normal.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
  - name: application
    image: application:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

#### 2.1.2. Replicação e Distribuição de Carga

Aplicações modernas devem suportar cargas variáveis, desde tráfego mínimo durante períodos de baixa utilização até picos substanciais durante eventos especiais ou horários de maior acesso. A execução de múltiplas réplicas idênticas de uma aplicação distribui a carga entre instâncias, melhorando capacidade agregada e proporcionando redundância que protege contra falhas individuais.

A gestão manual de réplicas apresenta desafios significativos. Operadores devem monitorar métricas de utilização continuamente, tomar decisões sobre quando escalar, iniciar ou terminar containers conforme necessário, e garantir que o balanceamento de carga distribua requisições apropriadamente entre todas as instâncias disponíveis. Este processo é propenso a erros, lento para responder a mudanças, e consome tempo operacional substancial.

O Kubernetes abstrai completamente esta complexidade através de recursos como Deployments e ReplicaSets, que garantem que um número especificado de réplicas esteja sempre executando, automaticamente substituindo instâncias que falham.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-application
  template:
    metadata:
      labels:
        app: web-application
    spec:
      containers:
      - name: application
        image: web-application:v1.2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

#### 2.1.3. Elasticidade e Autoescalamento

A demanda por recursos computacionais raramente permanece constante. Aplicações enfrentam padrões diários, semanais e sazonais de utilização, além de picos imprevisíveis causados por eventos externos. A capacidade de ajustar automaticamente o número de instâncias executando baseado em métricas observadas constitui requisito essencial para otimizar custos enquanto mantém performance aceitável.

O dimensionamento manual requer monitoramento constante e intervenções frequentes, resultando em escolhas subótimas entre provisionamento excessivo (desperdício de recursos) e provisionamento insuficiente (degradação de performance). O Kubernetes provê Horizontal Pod Autoscaler (HPA) e Vertical Pod Autoscaler (VPA) que ajustam automaticamente réplicas e recursos alocados baseados em métricas de CPU, memória, ou métricas customizadas específicas da aplicação.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-application-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-application
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### 2.1.4. Controle de Recursos Computacionais

Em ambientes multi-tenant ou com múltiplas aplicações compartilhando infraestrutura, o controle apropriado de recursos computacionais previne que aplicações individuais monopolizem recursos e impactem negativamente outras cargas de trabalho. Containers sem restrições podem consumir toda CPU ou memória disponível em um nó, causando degradação sistêmica.

O Kubernetes implementa resource requests e resource limits que especificam respectivamente os recursos mínimos que um container necessita e o máximo que pode consumir. O scheduler utiliza requests para tomar decisões de placement, garantindo que nós possuam capacidade suficiente. Os limits são enforçados pela runtime de containers através de cgroups, prevenindo consumo excessivo.

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

#### 2.1.5. Aplicações Stateless vs Stateful

A distinção entre aplicações stateless e stateful profundamente influencia estratégias de orquestração. Aplicações stateless não mantêm informações de sessão ou estado local persistente, permitindo que qualquer instância processe qualquer requisição. Esta característica facilita enormemente escalabilidade horizontal e recuperação de falhas, pois novas instâncias podem ser iniciadas instantaneamente sem necessidade de migrar ou sincronizar estado.

Aplicações stateful, por outro lado, mantêm dados locais críticos que devem persistir além do ciclo de vida de containers individuais. Bancos de dados, sistemas de cache, e filas de mensagens constituem exemplos comuns. O gerenciamento de aplicações stateful requer considerações adicionais incluindo armazenamento persistente, identidades de rede estáveis, e ordem garantida de inicialização e terminação.

O Kubernetes provê StatefulSets especificamente projetados para aplicações stateful, oferecendo garantias que Deployments tradicionais não fornecem. StatefulSets mantêm identidades consistentes para cada réplica, facilitam armazenamento persistente por instância, e controlam cuidadosamente a ordem de operações de escalamento.

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
      - name: database
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### 2.2. Complexidade de Aplicações Multi-Container

Aplicações modernas frequentemente decompõem funcionalidade em múltiplos processos especializados executando em containers separados. Esta arquitetura de microsserviços oferece vantagens significativas incluindo escalabilidade independente de componentes, desenvolvimento e implantação desacoplados, e isolamento de falhas. No entanto, a coordenação de dezenas ou centenas de microsserviços interdependentes introduz complexidades operacionais substanciais.

#### 2.2.1. Descoberta de Serviços e Comunicação

Em ambientes dinâmicos onde containers são criados, destruídos e movidos entre nós continuamente, mecanismos estáticos de descoberta baseados em endereços IP ou hostnames tornam-se impraticáveis. Aplicações necessitam descobrir dinamicamente a localização de serviços dependentes e adaptar-se automaticamente a mudanças na topologia.

O Kubernetes implementa Services que provêm abstração estável sobre conjuntos dinâmicos de Pods. Cada Service recebe um endereço IP virtual estável e nome DNS que permanecem constantes independentemente de mudanças nas instâncias subjacentes. O kube-proxy em cada nó mantém regras de rede que encaminham tráfego destinado a Services para Pods backend apropriados.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

#### 2.2.2. Configuração e Gestão de Segredos

Aplicações modernas requerem configurações que variam entre ambientes (desenvolvimento, staging, produção) e informações sensíveis como credenciais de banco de dados, chaves de API, e certificados TLS. A incorporação direta destas informações em imagens de containers viola princípios de segurança e dificulta gestão de configurações.

O Kubernetes provê ConfigMaps para dados de configuração não-sensíveis e Secrets para informações confidenciais. Estes recursos permitem desacoplar configuração de código de aplicação, facilitando promoção de aplicações entre ambientes sem reconstrução de imagens.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  database-url: "postgresql://db.example.com:5432/appdb"
  feature-flags: |
    feature_a=true
    feature_b=false
---
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: admin
  password: s3cr3tP@ssw0rd
```

## 3. Arquitetura do Kubernetes

### 3.1. Visão Geral da Arquitetura Distribuída

A arquitetura do Kubernetes fundamenta-se em um modelo cliente-servidor distribuído que separa claramente o plano de controle (control plane) dos nós de trabalho (worker nodes). O plano de controle gerencia o estado do cluster e toma decisões de orquestração, enquanto os nós de trabalho executam as cargas de trabalho containerizadas. Esta separação permite escalabilidade horizontal tanto do plano de controle quanto da capacidade computacional, além de facilitar alta disponibilidade através de replicação de componentes críticos.

Um cluster Kubernetes mínimo consiste em pelo menos um nó de plano de controle e um nó de trabalho. Em ambientes de produção, a prática recomendada envolve múltiplos nós de plano de controle (tipicamente três ou cinco para tolerância a falhas bizantinas) distribuídos através de zonas de disponibilidade distintas, e número variável de nós de trabalho dimensionado conforme requisitos de capacidade.

A comunicação entre componentes ocorre primariamente através do kube-apiserver, que expõe a API REST do Kubernetes. Todos os componentes, sejam no plano de controle ou nos nós de trabalho, interagem com o cluster exclusivamente através desta API, estabelecendo o apiserver como ponto central de coordenação. Esta arquitetura facilita extensibilidade através de custom resources e controllers, além de simplificar auditoria e aplicação de políticas de segurança.

### 3.2. Componentes do Plano de Controle

O plano de controle executa processos fundamentais incluindo o servidor de API Kubernetes, scheduler, controladores de recursos principais, e armazenamento do estado do cluster. Estes componentes trabalham colaborativamente para garantir que o estado atual do cluster convirja continuamente ao estado desejado especificado pelos usuários.

#### 3.2.1. kube-apiserver

O kube-apiserver constitui o componente central do plano de controle, servindo como frontend para a API do Kubernetes. Todas as operações no cluster, seja de usuários através do kubectl, de componentes internos, ou de integrações externas, são processadas através do apiserver. O kube-apiserver é projetado para escalar horizontalmente, permitindo execução de múltiplas instâncias com balanceamento de tráfego entre elas.

O apiserver implementa múltiplas responsabilidades críticas:

- **Validação e admissão**: Valida requests contra schemas da API, executa admission controllers que podem mutar ou rejeitar requests
- **Autenticação e autorização**: Autentica identidades através de múltiplos mecanismos (certificados, tokens, OIDC) e autoriza operações baseado em políticas RBAC
- **Persistência**: Coordena persistência de estado no etcd através de operações atômicas
- **Watching e notificações**: Mantém conexões de longa duração (watches) que notificam clientes sobre mudanças em recursos
- **API aggregation**: Permite que APIs customizadas sejam expostas através do mesmo endpoint

O apiserver não mantém estado local, derivando todo estado do etcd. Esta característica permite que instâncias sejam adicionadas ou removidas livremente para atender demandas de carga ou manutenção.

```bash
# Exemplo de interação com o apiserver através do kubectl
kubectl get pods --v=8  # Mostra detalhes da comunicação HTTP

# Acesso direto à API REST
curl -k -H "Authorization: Bearer ${TOKEN}" \
  https://kubernetes-api-server:6443/api/v1/namespaces/default/pods
```

#### 3.2.2. etcd

O etcd serve como armazenamento consistente e altamente disponível de chave-valor utilizado como repositório de backup para todos os dados do cluster Kubernetes. Originalmente desenvolvido pela CoreOS e posteriormente doado à CNCF, o etcd implementa o algoritmo de consenso Raft para garantir consistência distribuída mesmo em presença de falhas de rede ou nós.

O etcd armazena toda a configuração do cluster, incluindo definições de Pods, Services, ConfigMaps, Secrets, e estado operacional. A performance e confiabilidade do etcd impactam diretamente a disponibilidade e responsividade do cluster inteiro. Por esta razão, o etcd deve ser executado em hardware com baixa latência de disco (preferencialmente SSDs) e backups regulares constituem prática operacional essencial.

```bash
# Verificar saúde do etcd
kubectl exec -it -n kube-system etcd-control-plane -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Listar membros do cluster etcd
kubectl exec -it -n kube-system etcd-control-plane -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

O etcd expõe uma API gRPC que o kube-apiserver utiliza para todas as operações de leitura e escrita. Apenas o apiserver comunica diretamente com o etcd; outros componentes interagem com o cluster exclusivamente através da API do Kubernetes. Este isolamento simplifica segurança e permite que o etcd seja executado em rede separada com regras de firewall restritivas.

#### 3.2.3. kube-scheduler

O kube-scheduler observa Pods recém-criados sem nó atribuído e seleciona um nó apropriado para execução, considerando requisitos individuais e coletivos de recursos, restrições de hardware/software/políticas, e regras de afinidade e anti-afinidade. O processo de scheduling ocorre em duas fases: filtragem e pontuação.

Na fase de filtragem, o scheduler elimina nós que não satisfazem requisitos do Pod. Predicates incluem:

- **Recursos suficientes**: O nó possui CPU e memória disponíveis para satisfazer resource requests do Pod
- **Seletores de nó**: O Pod especifica nodeSelector ou nodeAffinity que o nó deve satisfazer
- **Taints e tolerations**: O Pod tolera quaisquer taints presentes no nó
- **Volumes**: Volumes requeridos podem ser montados no nó
- **Topologia**: Restrições de topologia (zona, região) são satisfeitas

Na fase de pontuação, o scheduler atribui scores aos nós restantes baseado em diversos fatores:

- **Balanceamento de recursos**: Preferência por nós com utilização equilibrada de CPU e memória
- **Afinidade**: Proximidade a outros Pods conforme especificado por regras de afinidade
- **Espalhamento**: Distribuição de Pods através de zonas ou nós para alta disponibilidade
- **Imagem já presente**: Preferência por nós que já possuem a imagem do container

O nó com maior score é selecionado. Se múltiplos nós empatam, um é escolhido aleatoriamente para distribuir carga uniformemente.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scheduled-pod
spec:
  # Requisitos de resources influenciam decisões de scheduling
  containers:
  - name: application
    image: application:latest
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
  
  # Node selector restringe nós elegíveis
  nodeSelector:
    disktype: ssd
  
  # Affinity rules influenciam preferências de scheduling
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - application
          topologyKey: kubernetes.io/hostname
```

#### 3.2.4. kube-controller-manager

O kube-controller-manager executa múltiplos controllers que implementam comportamentos da API do Kubernetes. Cada controller é um loop de controle que observa o estado atual do cluster através do apiserver e executa ações para mover o estado atual em direção ao estado desejado. Logicamente, cada controller constitui um processo separado, mas por simplicidade operacional, todos são compilados em um binário único executando como um processo.

Controllers importantes incluem:

- **Node Controller**: Monitora saúde dos nós e marca nós como não disponíveis quando falhas são detectadas
- **Replication Controller/ReplicaSet Controller**: Garante que o número correto de Pods esteja executando para cada ReplicaSet
- **Endpoints Controller**: Popula objetos Endpoints conectando Services aos Pods
- **Service Account & Token Controllers**: Cria contas de serviço padrão e tokens de API para novos namespaces
- **Deployment Controller**: Gerencia rollouts e rollbacks de Deployments
- **StatefulSet Controller**: Gerencia StatefulSets, garantindo identidades e ordem apropriadas
- **DaemonSet Controller**: Garante que Pods especificados executam em todos (ou alguns) nós
- **Job Controller**: Gerencia Jobs que executam até conclusão
- **CronJob Controller**: Gerencia Jobs agendados periodicamente

Cada controller opera independentemente, utilizando watches na API para receber notificações eficientes sobre mudanças em recursos relevantes. Esta arquitetura baseada em reconciliação garante que o sistema eventualmente convirja ao estado desejado mesmo em presença de falhas transitórias ou conflitos.

```yaml
# Exemplo de ReplicaSet gerenciado por controller
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: application-replicaset
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
      - name: application
        image: application:v1.0.0
```

O ReplicaSet Controller continuamente monitora este resource e garante que exatamente 3 Pods correspondentes ao template estejam executando. Se Pods são deletados ou falham, o controller cria novos. Se réplicas extras existem, o controller as remove.

#### 3.2.5. cloud-controller-manager

O cloud-controller-manager incorpora lógica de controle específica de provedores de nuvem, permitindo que clusters se integrem com APIs de provedores e separando componentes que interagem com plataformas cloud daqueles que interagem apenas com o cluster. Este componente executa apenas em clusters hospedados em infraestrutura de nuvem pública ou privada que oferece integrações nativas.

O cloud-controller-manager executa vários controllers específicos de cloud:

- **Node Controller**: Verifica com o provedor cloud se nós que pararam de reportar foram deletados na nuvem
- **Route Controller**: Configura rotas apropriadas na infraestrutura cloud
- **Service Controller**: Cria, atualiza e deleta load balancers de provedor cloud
- **Volume Controller**: Cria, anexa e monta volumes de armazenamento cloud

Esta separação permite que o core do Kubernetes permaneça agnóstico a provedores, enquanto vendors podem desenvolver e manter suas próprias integrações sem modificar o código central do Kubernetes.

### 3.3. Componentes dos Nós de Trabalho

Os nós de trabalho (worker nodes) executam as cargas de trabalho containerizadas e fornecem o ambiente de execução necessário para Pods. Cada nó executa componentes essenciais que comunicam com o plano de controle e gerenciam containers localmente.

#### 3.3.1. kubelet

O kubelet constitui o agente primário que executa em cada nó de trabalho, responsável por garantir que containers estejam executando em Pods conforme especificado. O kubelet não gerencia containers que não foram criados pelo Kubernetes. Este componente observa PodSpecs (especificações de Pods) fornecidas através do apiserver e garante que os containers descritos estejam executando e saudáveis.

Responsabilidades principais do kubelet incluem:

- **Registro de nó**: Registra o nó com o apiserver, tornando-o disponível para scheduling
- **Gestão de Pods**: Inicia, para e monitora containers especificados em PodSpecs
- **Health checking**: Executa liveness e readiness probes, reportando resultados ao apiserver
- **Coleta de métricas**: Expõe métricas de recursos (CPU, memória) através da API de métricas
- **Gestão de volumes**: Monta volumes especificados e os disponibiliza para containers
- **Image pulling**: Baixa imagens de container de registries conforme necessário

O kubelet comunica com a container runtime através do Container Runtime Interface (CRI), uma interface padronizada que permite suporte a múltiplas runtimes (containerd, CRI-O, Docker via dockershim em versões antigas).

```yaml
# PodSpec que o kubelet processa
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
  - name: app-container
    image: application:latest
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: data
      mountPath: /data
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
  volumes:
  - name: data
    emptyDir: {}
```

O kubelet periodicamente reporta o status do nó e dos Pods ao apiserver através de heartbeats. Se o apiserver não recebe heartbeats de um nó por período configurável (padrão 40 segundos), o Node Controller marca o nó como não disponível e o scheduler para de agendar novos Pods naquele nó.

#### 3.3.2. kube-proxy

O kube-proxy mantém regras de rede em cada nó que permitem comunicação de rede para Pods, implementando a abstração de Service. O kube-proxy observa o apiserver para adições e remoções de objetos Service e Endpoints, atualizando regras de encaminhamento de rede conforme necessário.

O kube-proxy pode operar em diferentes modos:

- **iptables mode**: Utiliza regras iptables para encaminhar tráfego. Oferece melhor performance que userspace mode mas requer mais regras para clusters grandes
- **IPVS mode**: Utiliza Linux IPVS (IP Virtual Server) para balanceamento de carga em kernel space. Oferece melhor performance e mais algoritmos de balanceamento para clusters muito grandes
- **userspace mode**: Modo legado onde kube-proxy age como proxy real, recebendo conexões e redirecionando para backends. Menos performante mas mais compatível

```bash
# Verificar modo de operação do kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"

# Exemplo de regras iptables criadas pelo kube-proxy
iptables -t nat -L KUBE-SERVICES -n | grep service-name
```

Quando um Service é criado, o kube-proxy em cada nó configura regras que interceptam tráfego destinado ao ClusterIP do Service e o distribuem entre os Pods backend. Esta implementação permite que Pods comuniquem com Services usando endereços IP estáveis, independentemente de onde os Pods backend estejam executando.

#### 3.3.3. Container Runtime

A container runtime é o software responsável por executar containers. Kubernetes suporta qualquer runtime que implemente o Container Runtime Interface (CRI), incluindo containerd, CRI-O, e Docker (através do dockershim em versões antigas, deprecado em 1.20, removido em 1.24).

O containerd tornou-se a runtime recomendada para a maioria dos casos de uso, oferecendo excelente equilíbrio entre performance, estabilidade e funcionalidades. O CRI-O é otimizado especificamente para Kubernetes e popular em ambientes OpenShift.

```bash
# Verificar runtime utilizada
kubectl get nodes -o wide

# Verificar containers executando com containerd
crictl ps

# Inspecionar imagens disponíveis
crictl images
```

A runtime interface com o kernel Linux para criar namespaces, cgroups, e outras primitivas de isolamento que implementam containers. A separação clara entre kubelet e runtime através do CRI permite que diferentes runtimes sejam substituídas sem modificação do Kubernetes.

### 3.4. Addons

Addons são Pods e Services que implementam funcionalidades de cluster essenciais. Enquanto componentes do plano de controle e dos nós são necessários para operação básica do cluster, addons provêm capacidades adicionais frequentemente essenciais para uso produtivo.

#### 3.4.1. DNS

O DNS do cluster (tipicamente CoreDNS) fornece resolução de nomes DNS para Services e Pods. Todos os containers criados no cluster são automaticamente configurados para utilizar este servidor DNS, permitindo descoberta de serviços através de nomes DNS ao invés de endereços IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  selector:
    app: backend
  ports:
  - port: 80
```

Este Service torna-se acessível via DNS em `backend-service.production.svc.cluster.local`, ou simplesmente `backend-service` para Pods no mesmo namespace.

#### 3.4.2. Web UI (Dashboard)

O Dashboard do Kubernetes fornece interface web para gestão e troubleshooting de recursos do cluster. Permite visualização de recursos, logs de containers, e execução de operações administrativas através de interface gráfica.

#### 3.4.3. Container Resource Monitoring

Soluções como Metrics Server coletam métricas de recursos de nós e containers, disponibilizando-as através da Metrics API. Estas métricas são essenciais para funcionalidade de autoescalamento e comandos como `kubectl top`.

#### 3.4.4. Cluster-level Logging

Agregadores de logs coletam logs de containers e os centralizam para análise e arquivamento. Soluções populares incluem Elasticsearch/Fluentd/Kibana (EFK stack) ou Loki/Promtail/Grafana.

## 4. Opções de Execução do Kubernetes

### 4.1. Kubernetes Gerenciado (Managed Kubernetes)

Provedores de nuvem pública oferecem serviços de Kubernetes gerenciado onde o provedor assume responsabilidade pela operação do plano de controle, incluindo atualizações, backups, e disponibilidade. Usuários focam exclusivamente em executar suas cargas de trabalho nos nós de trabalho.

Opções populares incluem:

- **Amazon EKS (Elastic Kubernetes Service)**: Integração profunda com serviços AWS como IAM, VPC, Load Balancers, e armazenamento EBS/EFS
- **Google GKE (Google Kubernetes Engine)**: Considerado referência em Kubernetes gerenciado, oferece autopilot mode onde Google gerencia também os nós
- **Azure AKS (Azure Kubernetes Service)**: Integração com Azure Active Directory, Azure Monitor, e outros serviços Azure
- **DigitalOcean Kubernetes**: Oferece simplicidade e preços competitivos para casos de uso mais simples
- **Linode Kubernetes Engine (LKE)**: Foco em simplicidade e performance

Vantagens do Kubernetes gerenciado:

- **Redução de complexidade operacional**: Provedor gerencia atualizações, backups do etcd, e disponibilidade do control plane
- **Integração nativa com serviços cloud**: Balanceadores de carga, armazenamento persistente, e redes funcionam automaticamente
- **Escalabilidade do plano de controle**: Provedor escala automaticamente componentes do control plane conforme necessário
- **Suporte e SLAs**: Disponibilidade de suporte técnico especializado e garantias de uptime

Desvantagens:

- **Custo**: Cobrança pela gestão do control plane além de custos de infraestrutura
- **Vendor lock-in**: Integrações específicas podem dificultar migração entre provedores
- **Controle limitado**: Acesso reduzido a configurações avançadas do plano de controle

```bash
# Exemplo: Criar cluster EKS na AWS
eksctl create cluster \
  --name production-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# Exemplo: Criar cluster GKE no Google Cloud
gcloud container clusters create production-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n1-standard-2 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 10
```

### 4.2. Kubernetes Auto-Gerenciado (Self-Managed)

Em implantações auto-gerenciadas, organizações assumem responsabilidade completa por todos os aspectos do cluster, incluindo plano de controle e nós de trabalho. Esta abordagem oferece controle máximo mas requer expertise operacional significativa.

Ferramentas populares para instalação:

- **kubeadm**: Ferramenta oficial para bootstrapping de clusters, implementa melhores práticas da comunidade
- **kops**: Kubernetes Operations, especializada em instalação em AWS mas suporta outros provedores
- **Kubespray**: Utiliza Ansible para deployment em infraestrutura existente, altamente customizável
- **Rancher**: Plataforma completa que facilita deployment e gestão de múltiplos clusters

```bash
# Exemplo básico com kubeadm

# No nó master
kubeadm init --pod-network-cidr=10.244.0.0/16

# Configurar kubectl para usuário
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Instalar network plugin (ex: Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Nos nós worker
kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Vantagens do auto-gerenciamento:

- **Controle total**: Acesso completo a todas as configurações e componentes
- **Flexibilidade**: Customização ilimitada para requisitos específicos
- **Independência de vendor**: Não depende de provedor específico
- **Potencial redução de custos**: Elimina custos de gestão de provedor

Desvantagens:

- **Complexidade operacional**: Requer expertise especializada para manutenção
- **Responsabilidade completa**: Organização assume todos os riscos de disponibilidade e segurança
- **Overhead de manutenção**: Atualizações, backups, e troubleshooting requerem recursos dedicados

### 4.3. Ambientes Locais para Desenvolvimento

Para aprendizado, desenvolvimento local, e testes, existem ferramentas que executam clusters Kubernetes completos em máquinas individuais.

#### 4.3.1. kind (Kubernetes IN Docker)

O kind executa clusters Kubernetes utilizando containers Docker como "nós". É extremamente rápido para criar e destruir clusters, ideal para CI/CD e desenvolvimento local.

```bash
# Instalação do kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Criar cluster simples
kind create cluster --name dev-cluster

# Criar cluster com configuração customizada
cat <<EOF | kind create cluster --name multi-node --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

# Listar clusters
kind get clusters

# Deletar cluster
kind delete cluster --name dev-cluster
```

#### 4.3.2. minikube

O minikube executa cluster Kubernetes de nó único em máquina local, suportando múltiplos drivers (Docker, VirtualBox, KVM, Hyper-V).

```bash
# Instalação do minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Iniciar cluster
minikube start --cpus 4 --memory 8192

# Habilitar addons
minikube addons enable ingress
minikube addons enable metrics-server

# Acessar dashboard
minikube dashboard

# Parar cluster
minikube stop

# Deletar cluster
minikube delete
```

#### 4.3.3. k3s

O k3s é distribuição leve do Kubernetes projetada para edge computing, IoT, e CI. Remove componentes opcionais, resultando em binário único de menos de 100MB.

```bash
# Instalação do k3s
curl -sfL https://get.k3s.io | sh -

# Verificar status
sudo systemctl status k3s

# Acessar cluster
sudo k3s kubectl get nodes

# Configurar kubectl regular
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

## 5. Interação com Clusters Kubernetes

### 5.1. kubectl - Interface de Linha de Comando

O kubectl constitui a ferramenta primária para interagir com clusters Kubernetes, fornecendo interface de linha de comando para operações de gestão, deployment, troubleshooting e monitoramento.

#### 5.1.1. Instalação e Configuração

```bash
# Instalação no Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verificar instalação
kubectl version --client

# Verificar conectividade com cluster
kubectl cluster-info

# Verificar configuração atual
kubectl config view

# Listar contextos disponíveis
kubectl config get-contexts

# Trocar contexto
kubectl config use-context <context-name>
```

#### 5.1.2. Comandos Fundamentais

```bash
# Gerenciamento de recursos
kubectl get <resource>              # Listar recursos
kubectl describe <resource> <name>  # Descrever recurso específico
kubectl create -f <file.yaml>       # Criar recurso
kubectl apply -f <file.yaml>        # Criar ou atualizar recurso
kubectl delete <resource> <name>    # Deletar recurso
kubectl edit <resource> <name>      # Editar recurso interativamente

# Exemplos práticos
kubectl get pods --all-namespaces
kubectl get deployments -n production
kubectl describe pod my-pod -n default
kubectl logs my-pod -n default
kubectl logs my-pod -c container-name --follow
kubectl exec -it my-pod -- /bin/bash
kubectl port-forward pod/my-pod 8080:80

# Debugging e troubleshooting
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes
kubectl top pods -n production
kubectl explain pod.spec.containers
```

#### 5.1.3. Gestão de Contextos e Namespaces

```bash
# Configurar namespace padrão para contexto
kubectl config set-context --current --namespace=production

# Criar namespace
kubectl create namespace development

# Aplicar recursos em namespace específico
kubectl apply -f deployment.yaml -n production

# Deletar namespace (deleta todos os recursos)
kubectl delete namespace development
```

### 5.2. Ferramentas Visuais

#### 5.2.1. Lens - Kubernetes IDE

Lens oferece interface gráfica completa para gestão de clusters Kubernetes, proporcionando visualização intuitiva de recursos, métricas em tempo real, terminal integrado, e editor de manifestos.

Características principais:

- **Multi-cluster management**: Gerenciar múltiplos clusters simultaneamente
- **Métricas integradas**: Visualização de CPU, memória e outros recursos sem configuração adicional
- **Terminal integrado**: Acesso rápido a shell de containers
- **Editor de manifestos**: Edição visual de recursos com validação
- **Extensibilidade**: Sistema de plugins para funcionalidades adicionais

```bash
# Instalação no Linux (via snap)
snap install kontena-lens --classic

# Adicionar cluster
# Lens detecta automaticamente clusters no kubeconfig
```

#### 5.2.2. k9s - Terminal UI

O k9s fornece interface de terminal interativa e eficiente para gerenciar clusters Kubernetes, combinando velocidade de linha de comando com usabilidade de interface gráfica.

```bash
# Instalação
curl -sS https://webi.sh/k9s | sh

# Executar k9s
k9s

# Comandos dentro do k9s
# :pods          - Listar pods
# :deployments   - Listar deployments
# :services      - Listar services
# /              - Filtrar recursos
# l              - Ver logs
# d              - Descrever recurso
# e              - Editar recurso
# Ctrl+d         - Deletar recurso
```

#### 5.2.3. Kubernetes Dashboard

Dashboard oficial do Kubernetes fornece interface web para visualização e gestão de recursos do cluster.

```bash
# Instalar dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Criar service account e obter token
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin

# Obter token
kubectl -n kubernetes-dashboard create token dashboard-admin

# Iniciar proxy
kubectl proxy

# Acessar em: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## 6. Conclusões

O Kubernetes revolucionou a forma como aplicações modernas são implantadas e operadas, estabelecendo padrões de indústria para orquestração de containers e consolidando-se como tecnologia fundamental para arquiteturas cloud-native. A capacidade de abstrair infraestrutura subjacente enquanto fornece primitivas poderosas para gestão de aplicações distribuídas democratizou acesso a práticas operacionais que anteriormente estavam restritas a organizações com recursos substanciais.

A arquitetura distribuída do Kubernetes, fundamentada na separação entre plano de controle e nós de trabalho, combinada com modelo declarativo de gestão de configuração, oferece fundação robusta para construção de sistemas resilientes e escaláveis. Os componentes essenciais - apiserver, etcd, scheduler, controller-manager nos planos de controle, e kubelet, kube-proxy nos nós de trabalho - colaboram através de interfaces bem definidas para garantir convergência contínua do estado atual ao estado desejado especificado pelos usuários.

A versatilidade do Kubernetes manifesta-se através de suas múltiplas opções de deployment, desde serviços gerenciados oferecidos por todos os principais provedores de nuvem até instalações completamente customizadas em infraestrutura própria. Para desenvolvimento local e aprendizado, ferramentas como kind, minikube e k3s removem barreiras de entrada, permitindo que desenvolvedores experimentem com Kubernetes em máquinas pessoais sem necessidade de infraestrutura complexa ou custos de nuvem.

A adoção do Kubernetes requer consideração cuidadosa de trade-offs entre complexidade e benefícios. Enquanto o sistema oferece capacidades extraordinárias de escalabilidade, resiliência e automação, a curva de aprendizado é significativa e a operação adequada demanda expertise especializada. Organizações devem avaliar criticamente se a escala e complexidade de suas aplicações justificam a infraestrutura e overhead operacional associados ao Kubernetes, considerando alternativas mais simples quando apropriado.

O ecossistema em torno do Kubernetes continua evoluindo rapidamente, com inovações constantes em áreas como service mesh (Istio, Linkerd), serverless (Knative), gerenciamento de políticas (OPA), segurança (Falco, Cilium), e observabilidade (Prometheus, Grafana, Jaeger). A governança provida pela CNCF e contribuições de centenas de organizações garantem que o Kubernetes permanecerá relevante e continuará se adaptando às necessidades emergentes de aplicações cloud-native nas próximas décadas.

## 7. Referências Bibliográficas

### 7.1. Documentação Oficial

- Kubernetes Documentation. "Kubernetes Documentation". Kubernetes.io. Disponível em: https://kubernetes.io/docs/home/

- Kubernetes Documentation. "Kubernetes Components". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/overview/components/

- Kubernetes Documentation. "Cluster Architecture". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/architecture/

- Kubernetes API Reference. "Kubernetes API Reference Documentation". Kubernetes.io. Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/

### 7.2. Cloud Native Computing Foundation

- Cloud Native Computing Foundation. "CNCF Official Website". CNCF.io. Disponível em: https://www.cncf.io/

- CNCF. "Kubernetes Project". CNCF Projects. Disponível em: https://www.cncf.io/projects/kubernetes/

- CNCF. "Cloud Native Landscape". CNCF Landscape. Disponível em: https://landscape.cncf.io/

### 7.3. Ferramentas e Projetos

- etcd Documentation. "etcd Official Documentation". etcd.io. Disponível em: https://etcd.io/docs/

- kind Project. "kind - Kubernetes IN Docker". kind.sigs.k8s.io. Disponível em: https://kind.sigs.k8s.io/

- minikube Project. "minikube Documentation". minikube.sigs.k8s.io. Disponível em: https://minikube.sigs.k8s.io/docs/

- k3s Documentation. "k3s - Lightweight Kubernetes". k3s.io. Disponível em: https://k3s.io/

- kubectl Documentation. "kubectl Reference Documentation". Kubernetes.io. Disponível em: https://kubernetes.io/docs/reference/kubectl/

### 7.4. Kubernetes Gerenciado

- Amazon Web Services. "Amazon Elastic Kubernetes Service (EKS)". AWS.amazon.com. Disponível em: https://aws.amazon.com/eks/

- Google Cloud. "Google Kubernetes Engine (GKE)". Cloud.google.com. Disponível em: https://cloud.google.com/kubernetes-engine

- Microsoft Azure. "Azure Kubernetes Service (AKS)". Azure.microsoft.com. Disponível em: https://azure.microsoft.com/en-us/services/kubernetes-service/

### 7.5. Instalação e Deployment

- Kubernetes Documentation. "Installing kubeadm". Kubernetes.io. Disponível em: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

- kops Project. "Kubernetes Operations (kops)". kops.sigs.k8s.io. Disponível em: https://kops.sigs.k8s.io/

- Kubespray Project. "Kubespray - Deploy a Production Ready Kubernetes Cluster". Kubespray GitHub. Disponível em: https://github.com/kubernetes-sigs/kubespray

## 8. Apêndices

### Apêndice A: Comandos kubectl Essenciais

```bash
# Informações do cluster
kubectl cluster-info
kubectl version
kubectl api-resources
kubectl api-versions

# Gerenciamento de recursos
kubectl get all --all-namespaces
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pods -l app=nginx
kubectl get pods --field-selector status.phase=Running

# Criação e atualização
kubectl create deployment nginx --image=nginx:latest
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.21
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx

# Debugging
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>
kubectl exec -it <pod-name> -- /bin/bash
kubectl port-forward <pod-name> 8080:80
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl top nodes
kubectl top pods

# Configuração
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<namespace>

# Gestão de recursos via arquivo
kubectl apply -f manifest.yaml
kubectl apply -f ./directory/
kubectl apply -k ./kustomize-directory/
kubectl delete -f manifest.yaml
kubectl diff -f manifest.yaml

# Úteis para troubleshooting
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning
kubectl explain pod.spec.containers
kubectl auth can-i create deployments
kubectl drain <node-name> --ignore-daemonsets
kubectl cordon <node-name>
kubectl uncordon <node-name>
```

### Apêndice B: Exemplo de Cluster Multi-Nó com kind

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: production-like-cluster
nodes:
# Control plane node
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP

# Worker nodes
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "workload-type=general"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "workload-type=general"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "workload-type=compute-intensive"
```

```bash
# Criar cluster com configuração
kind create cluster --config kind-config.yaml

# Verificar nós criados
kubectl get nodes

# Instalar Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Aguardar ingress controller estar pronto
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Apêndice C: Estrutura de Diretórios Recomendada para Manifestos

```
kubernetes-manifests/
├── base/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── replica-count.yaml
│   │   └── resource-limits.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── replica-count.yaml
│   │   └── resource-limits.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── replica-count.yaml
│       ├── resource-limits.yaml
│       └── hpa.yaml
└── README.md
```

### Apêndice D: Troubleshooting Comum

#### D.1. Pod em CrashLoopBackOff

```bash
# Ver logs do container
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Descrever pod para ver eventos
kubectl describe pod <pod-name>

# Verificar se imagem existe
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'

# Verificar probes
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].livenessProbe}'
```

#### D.2. Pod em ImagePullBackOff

```bash
# Verificar secret de registry
kubectl get secrets
kubectl describe secret <registry-secret>

# Verificar eventos do pod
kubectl describe pod <pod-name> | grep -A 10 Events

# Testar pull manual da imagem
docker pull <image-name>
```

#### D.3. Service Não Acessível

```bash
# Verificar se service existe
kubectl get service <service-name>

# Verificar endpoints do service
kubectl get endpoints <service-name>

# Verificar labels dos pods
kubectl get pods --show-labels

# Verificar selector do service
kubectl get service <service-name> -o jsonpath='{.spec.selector}'

# Testar conectividade de dentro do cluster
kubectl run test --image=busybox -it --rm -- wget -O- <service-name>:80
```

### Apêndice E: Glossário e Termos Técnicos

**API Server (kube-apiserver)**: Componente do plano de controle que expõe a API do Kubernetes, servindo como frontend para todas as operações do cluster.

**CNCF (Cloud Native Computing Foundation)**: Organização que hospeda e promove projetos open source cloud-native, incluindo Kubernetes.

**ConfigMap**: Objeto do Kubernetes utilizado para armazenar dados de configuração não-confidenciais em pares chave-valor.

**Container**: Unidade padrão de software que empacota código e todas as suas dependências para execução consistente entre ambientes.

**Container Runtime**: Software responsável por executar containers, como containerd, CRI-O ou Docker.

**Container Runtime Interface (CRI)**: Interface padronizada entre kubelet e container runtime.

**Control Plane**: Conjunto de componentes que gerenciam o estado do cluster e tomam decisões de orquestração (apiserver, scheduler, controller-manager, etcd).

**Controller**: Loop de controle que observa o estado do cluster e executa ações para mover o estado atual em direção ao estado desejado.

**CrashLoopBackOff**: Estado de um Pod que está falhando repetidamente ao iniciar e sendo reiniciado pelo Kubernetes.

**DaemonSet**: Recurso que garante que uma cópia de um Pod execute em todos (ou alguns) nós do cluster.

**Declarative Configuration**: Abordagem onde o usuário especifica o estado desejado e o sistema trabalha para alcançá-lo, ao invés de especificar sequência de comandos.

**Deployment**: Recurso que fornece atualizações declarativas para Pods e ReplicaSets, gerenciando rollouts e rollbacks.

**Endpoint**: Objeto que rastreia endereços IP e portas de Pods que implementam um Service.

**etcd**: Armazenamento de chave-valor distribuído e consistente utilizado como repositório de dados do Kubernetes.

**Health Check**: Verificação periódica do estado de um container através de liveness ou readiness probes.

**Horizontal Scaling**: Adicionar ou remover réplicas de uma aplicação para ajustar capacidade.

**ImagePullBackOff**: Estado indicando falha ao baixar imagem de container do registry.

**Ingress**: Recurso que gerencia acesso externo a Services no cluster, tipicamente HTTP/HTTPS.

**K8s**: Abreviação de Kubernetes (K + 8 letras + s).

**kind (Kubernetes IN Docker)**: Ferramenta para executar clusters Kubernetes locais usando containers Docker como nós.

**kube-controller-manager**: Componente que executa controllers responsáveis por comportamentos da API do Kubernetes.

**kube-proxy**: Componente executando em cada nó que mantém regras de rede para implementar Services.

**kube-scheduler**: Componente que seleciona nós para execução de Pods recém-criados.

**kubectl**: Interface de linha de comando para interagir com clusters Kubernetes.

**Kubelet**: Agente executando em cada nó que garante containers estejam executando em Pods conforme especificado.

**Lens**: IDE gráfica para gerenciamento de clusters Kubernetes.

**Liveness Probe**: Verificação que determina se um container está executando corretamente e deve ser reiniciado se falhar.

**Manifest**: Arquivo (geralmente YAML) que descreve recursos do Kubernetes de forma declarativa.

**Metrics Server**: Componente que coleta métricas de recursos de nós e Pods.

**minikube**: Ferramenta que executa cluster Kubernetes de nó único em máquina local.

**Namespace**: Mecanismo para isolar grupos de recursos dentro de um único cluster.

**Node**: Máquina (física ou virtual) no cluster Kubernetes onde containers são executados.

**Node Affinity**: Regras que restringem quais nós podem executar um Pod baseado em labels.

**Orchestration**: Automação de deployment, gerenciamento, escalabilidade e networking de aplicações containerizadas.

**Pod**: Menor unidade deployável no Kubernetes, consistindo de um ou mais containers que compartilham recursos.

**Pod Affinity**: Regras que influenciam scheduling baseado em outros Pods já executando em nós.

**PodSpec**: Especificação declarativa de um Pod, incluindo containers, volumes, e outras configurações.

**Readiness Probe**: Verificação que determina se um container está pronto para receber tráfego.

**Reconciliation Loop**: Processo contínuo onde controllers comparam estado atual com estado desejado e tomam ações para convergência.

**Registry**: Repositório para armazenamento e distribuição de imagens de containers.

**Replica**: Instância individual de um Pod gerenciado por controller.

**ReplicaSet**: Recurso que mantém conjunto estável de Pods réplicas executando a qualquer momento.

**Resource Limits**: Quantidade máxima de recursos (CPU, memória) que um container pode consumir.

**Resource Requests**: Quantidade de recursos que um container declara necessitar, utilizada pelo scheduler.

**Rollback**: Reversão de deployment para versão anterior.

**Rollout**: Processo de deployment de nova versão de aplicação.

**Scheduler**: Componente que atribui Pods recém-criados a nós do cluster.

**Secret**: Objeto que armazena informações sensíveis como senhas, tokens ou chaves.

**Service**: Abstração que define política de acesso lógico a conjunto de Pods.

**Service Mesh**: Camada de infraestrutura dedicada que gerencia comunicação serviço-a-serviço.

**StatefulSet**: Recurso para gerenciar aplicações stateful que requerem identidades persistentes e armazenamento ordenado.

**Taint**: Propriedade de nó que repele Pods, a menos que os Pods tenham tolerations correspondentes.

**Toleration**: Propriedade de Pod que permite (mas não requer) scheduling em nós com taints correspondentes.

**Vertical Scaling**: Aumentar ou diminuir recursos (CPU, memória) alocados a instâncias existentes.

**Volume**: Diretório acessível a containers em um Pod, potencialmente persistente além do ciclo de vida do Pod.

**Watch**: Mecanismo de streaming da API que notifica clientes sobre mudanças em recursos.

**Worker Node**: Nó no cluster Kubernetes que executa cargas de trabalho containerizadas.

**YAML (YAML Ain't Markup Language)**: Formato de serialização de dados legível por humanos, comumente utilizado para manifestos Kubernetes.
