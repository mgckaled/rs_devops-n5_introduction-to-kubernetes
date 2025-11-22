<!-- markdownlint-disable -->

# Bloco B - Orquestrando Containers com Pods, ReplicaSets, Deployments e Services

## Resumo Executivo

A orquestração de containers no Kubernetes fundamenta-se em abstrações hierárquicas que progressivamente adicionam capacidades de gestão, resiliência e automação à execução de cargas de trabalho containerizadas. Este documento explora sistematicamente os recursos fundamentais do Kubernetes utilizados para implantar e gerenciar aplicações, começando pelo Pod como unidade atômica de deployment, avançando através de controladores como ReplicaSet que garantem disponibilidade através de replicação, e culminando em Deployments que adicionam capacidades sofisticadas de gestão de ciclo de vida e versionamento.

A transição de abordagens imperativas para declarativas constitui tema central na operação efetiva do Kubernetes. Enquanto comandos imperativos permitem experimentação rápida e prototipagem, a gestão declarativa através de manifestos YAML versionados representa a prática recomendada para ambientes de produção, proporcionando reprodutibilidade, rastreabilidade e capacidade de automação através de pipelines de CI/CD. A organização lógica de recursos através de namespaces facilita governança, isolamento e gestão de permissões em clusters compartilhados entre múltiplas equipes ou ambientes.

O conceito de efemeridade de Pods - sua natureza descartável e transitória - motiva a necessidade de controladores que garantam disponibilidade contínua através de replicação automática. ReplicaSets implementam esta funcionalidade, mantendo número especificado de réplicas de Pods executando a qualquer momento, recriando automaticamente instâncias que falham. No entanto, ReplicaSets sozinhos mostram-se inadequados para cenários reais de atualização de aplicações, levando à criação de Deployments que orquestram atualizações controladas através de estratégias como rolling updates, proporcionando atualizações sem downtime e capacidade de rollback em caso de problemas.

A exposição de aplicações containerizadas para comunicação interna ou externa requer abstrações de rede que desacoplem clientes dos Pods efêmeros subjacentes. Services no Kubernetes provêm endpoints estáveis com descoberta automática através de DNS e balanceamento de carga integrado, permitindo que aplicações comuniquem através de nomes simbólicos independentemente de mudanças na topologia dos Pods. O tipo ClusterIP, padrão no Kubernetes, oferece conectividade interna ao cluster, formando a base sobre a qual outros tipos de Services constroem funcionalidades adicionais.

## 1. Introdução e Conceitos

### 1.1. Hierarquia de Abstrações no Kubernetes

O Kubernetes organiza recursos em hierarquia conceitual onde cada nível adiciona capacidades sobre o anterior, formando camadas de abstração que simplificam progressivamente a gestão de aplicações distribuídas. Na base desta hierarquia encontram-se containers individuais, encapsulados em Pods que formam a unidade atômica de deployment. Controllers como ReplicaSets e Deployments constroem sobre Pods, adicionando lógica de replicação, recuperação de falhas e gestão de versões. Services complementam esta arquitetura fornecendo camada de rede que abstrai a localização dinâmica dos Pods.

Esta organização hierárquica reflete princípios de engenharia de software bem estabelecidos, onde abstrações de alto nível escondem complexidade implementacional enquanto expõem interfaces simples e expressivas. Desenvolvedores e operadores interagem primariamente com abstrações de alto nível como Deployments e Services, permitindo que o Kubernetes gerencie automaticamente recursos de nível inferior como Pods e endpoints de rede.

### 1.2. Abordagens Imperativas versus Declarativas

A gestão de recursos no Kubernetes pode ser realizada através de duas abordagens fundamentalmente diferentes: imperativa e declarativa. A abordagem imperativa, baseada em comandos diretos através do kubectl, instrui o cluster a executar ações específicas imediatamente. Por exemplo, `kubectl run nginx --image=nginx` cria um Pod executando nginx através de comando único que especifica explicitamente a ação desejada.

```bash
# Exemplos de comandos imperativos
kubectl run nginx --image=nginx:latest
kubectl expose pod nginx --port=80 --type=ClusterIP
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.21
```

A abordagem declarativa, por outro lado, baseia-se em manifestos que descrevem o estado desejado do sistema. Administradores criam arquivos YAML especificando características de recursos, e o Kubernetes trabalha continuamente para garantir que o estado atual convirja ao estado desejado especificado. Esta abordagem fundamental diferencia Kubernetes de ferramentas de automação tradicionais baseadas em scripts imperativos.

```yaml
# Exemplo de manifesto declarativo
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

A abordagem declarativa oferece vantagens significativas para ambientes de produção. Manifestos podem ser versionados em sistemas de controle de versão como Git, proporcionando histórico completo de mudanças, capacidade de revisão através de pull requests, e reversão trivial a configurações anteriores conhecidas. A automação através de pipelines de CI/CD torna-se direta, onde commits em repositórios automaticamente acionam aplicações de manifestos atualizados. A documentação torna-se intrínseca, pois os manifestos servem simultaneamente como especificação e implementação do estado desejado.

### 1.3. Namespaces como Mecanismo de Organização

Namespaces fornecem mecanismo de isolamento lógico que permite particionar um cluster Kubernetes em múltiplos clusters virtuais. Cada namespace provê escopo separado para nomes de recursos, permitindo que equipes diferentes utilizem o mesmo cluster sem conflitos de nomenclatura. Resources como Pods, Services e Deployments existem dentro de namespaces específicos, enquanto recursos de cluster como Nodes e PersistentVolumes operam no escopo global do cluster.

```bash
# Operações com namespaces
kubectl get namespaces
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Listar recursos em namespace específico
kubectl get pods -n development
kubectl get all -n production

# Definir namespace padrão para contexto atual
kubectl config set-context --current --namespace=development
```

A organização através de namespaces facilita implementação de políticas de governança e segurança. ResourceQuotas podem limitar consumo agregado de recursos por namespace, prevenindo que uma equipe monopolize recursos do cluster. NetworkPolicies podem isolar tráfego de rede entre namespaces, implementando segmentação de rede ao nível de aplicação. Role-Based Access Control (RBAC) utiliza namespaces como unidade de escopo para permissões, permitindo controle granular sobre quem pode acessar ou modificar recursos.

```yaml
# Manifesto com namespace explícito
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
  namespace: production
  labels:
    app: application
    environment: production
spec:
  containers:
  - name: application
    image: application:v1.0.0
```

Namespaces padrão criados automaticamente em clusters Kubernetes incluem:

- **default**: Namespace utilizado quando nenhum é especificado explicitamente
- **kube-system**: Contém recursos criados pelo sistema Kubernetes, como componentes do control plane
- **kube-public**: Automaticamente legível por todos os usuários, tipicamente usado para recursos que devem ser visíveis publicamente
- **kube-node-lease**: Contém objetos Lease associados a cada nó, utilizados para detecção de falhas de nós

## 2. Pods: Unidade Fundamental de Deployment

### 2.1. Conceito e Características de Pods

Pods constituem a menor unidade deployável no Kubernetes, representando uma ou mais containers executando em conjunto no mesmo nó e compartilhando recursos de rede e armazenamento. Diferentemente de containers individuais, Pods provêm contexto de execução compartilhado que permite co-localização de containers intimamente acoplados que necessitam comunicar-se frequentemente ou compartilhar volumes de dados.

Cada Pod recebe endereço IP único no cluster, compartilhado por todos os containers dentro do Pod. Containers no mesmo Pod comunicam através de localhost, pois compartilham namespace de rede. Volumes definidos no Pod podem ser montados em múltiplos containers, facilitando compartilhamento de dados. Esta co-localização e compartilhamento de recursos tornam Pods apropriados para padrões de design como sidecar, ambassador e adapter, onde containers auxiliares complementam funcionalidade do container principal.

```yaml
# Pod básico com container único
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
```

### 2.2. Efemeridade e Ciclo de Vida

Pods são entidades efêmeras por design, significando que podem ser criados, destruídos e substituídos frequentemente durante operação normal do cluster. Esta efemeridade contrasta fundamentalmente com modelos de infraestrutura tradicional onde servidores são tratados como "pets" com identidades persistentes e longevidade esperada. No paradigma Kubernetes, Pods são "cattle" - indistinguíveis, descartáveis e facilmente substituíveis.

O ciclo de vida de um Pod progride através de fases distintas:

- **Pending**: Pod foi aceito pelo cluster mas um ou mais containers não foram criados ou iniciados
- **Running**: Pod foi vinculado a um nó e todos os containers foram criados, com pelo menos um container executando
- **Succeeded**: Todos os containers terminaram com sucesso e não serão reiniciados
- **Failed**: Todos os containers terminaram e pelo menos um terminou com falha
- **Unknown**: Estado do Pod não pode ser determinado, tipicamente devido a erro de comunicação com o nó

```bash
# Observar ciclo de vida de um Pod
kubectl get pod nginx-pod -w

# Ver ReplicaSets gerenciados pelo Deployment
kubectl get replicaset -l app=nginx

# Ver Pods
kubectl get pods -l app=nginx

# Atualizar imagem (imperativo)
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Atualizar declarativamente
# Editar deployment.yaml alterando image
kubectl apply -f deployment.yaml

# Verificar status do rollout
kubectl rollout status deployment/nginx-deployment

# Ver histórico de rollouts
kubectl rollout history deployment/nginx-deployment

# Ver detalhes de revisão específica
kubectl rollout history deployment/nginx-deployment --revision=2

# Ver logs de container em Pod
kubectl logs nginx-pod
kubectl logs nginx-pod -c nginx --follow

# Executar comando em container
kubectl exec -it nginx-pod -- /bin/bash
```

A efemeridade dos Pods implica que aplicações não devem depender de identidades específicas de Pods ou assumir que Pods individuais permanecerão disponíveis indefinidamente. Em vez disso, aplicações devem ser projetadas para tolerar falhas de Pods individuais, confiando em controllers para manter disponibilidade através de replicação.

### 2.3. Especificação de Recursos

A especificação apropriada de requisitos de recursos constitui prática essencial para operação eficiente de clusters Kubernetes. Resource requests indicam quantidade mínima de CPU e memória que um container necessita, utilizados pelo scheduler ao decidir em qual nó posicionar o Pod. Resource limits definem máximos que um container pode consumir, enforçados pela runtime através de cgroups do Linux.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-constrained-pod
spec:
  containers:
  - name: application
    image: application:latest
    resources:
      requests:
        cpu: 250m        # 250 millicores (0.25 cores)
        memory: 256Mi    # 256 Mebibytes
      limits:
        cpu: 1000m       # 1000 millicores (1 core)
        memory: 512Mi    # 512 Mebibytes
```

A unidade `m` para CPU representa millicores, onde 1000m equivale a 1 core completo. Valores de memória utilizam notação binária (Mi, Gi) ou decimal (M, G), onde 1Mi = 1024 * 1024 bytes e 1M = 1000 * 1000 bytes. A escolha entre notações afeta valores exatos mas raramente impacta funcionalidade significativamente.

Quando um container excede seu memory limit, é terminado com erro OOMKilled (Out Of Memory Killed). CPU, sendo recurso compressível, é throttled quando limites são excedidos, degradando performance mas não terminando o container. Requests garantem que nós tenham capacidade suficiente, enquanto limits previnem que containers monopolizem recursos e impactem outras cargas de trabalho.

### 2.4. Criação Imperativa de Pods

A criação imperativa de Pods através de linha de comando oferece abordagem rápida para experimentação e troubleshooting, embora não seja recomendada para uso em produção devido à falta de reprodutibilidade e rastreabilidade.

```bash
# Criar Pod com imagem específica
kubectl run nginx --image=nginx:1.21

# Criar Pod com porta exposta
kubectl run nginx --image=nginx:1.21 --port=80

# Criar Pod com labels
kubectl run nginx --image=nginx:1.21 --labels="app=nginx,tier=frontend"

# Criar Pod com resource requests e limits
kubectl run nginx --image=nginx:1.21 \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=500m,memory=256Mi'

# Criar Pod e expor via Service simultaneamente
kubectl run nginx --image=nginx:1.21 --port=80 --expose

# Criar Pod em namespace específico
kubectl run nginx --image=nginx:1.21 -n development

# Gerar manifesto YAML sem criar recurso
kubectl run nginx --image=nginx:1.21 --dry-run=client -o yaml > pod.yaml
```

O uso de `--dry-run=client -o yaml` permite gerar templates de manifestos rapidamente, combinando conveniência de linha de comando com benefícios de gestão declarativa. O manifesto gerado pode ser editado conforme necessário e commitado em repositório de controle de versão.

### 2.5. Port Forwarding e Acesso a Pods

Durante desenvolvimento e troubleshooting, frequentemente é necessário acessar Pods diretamente sem criar Services ou Ingress. O comando `kubectl port-forward` estabelece túnel seguro entre máquina local e Pod específico, permitindo acesso a portas de containers.

```bash
# Forward porta 8080 local para porta 80 do Pod
kubectl port-forward nginx-pod 8080:80

# Forward porta aleatória local para porta 80 do Pod
kubectl port-forward nginx-pod :80

# Forward múltiplas portas
kubectl port-forward nginx-pod 8080:80 8443:443

# Forward porta em namespace específico
kubectl port-forward -n production nginx-pod 8080:80

# Forward em background
kubectl port-forward nginx-pod 8080:80 &

# Acessar aplicação via port forward
curl http://localhost:8080
```

Port forwarding destina-se exclusivamente a uso em desenvolvimento e debugging. Para acesso em produção, Services e Ingress Controllers fornecem soluções apropriadas com balanceamento de carga, descoberta de serviços e configurações de rede adequadas.

### 2.6. Manifestos Declarativos de Pods

A gestão declarativa através de manifestos YAML representa a prática recomendada para ambientes de produção, proporcionando documentação auto-explicativa, versionamento através de Git, e capacidade de revisão e auditoria de mudanças.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-production
  namespace: production
  labels:
    app: nginx
    environment: production
    version: v1.21
  annotations:
    description: "NGINX web server for production frontend"
    owner: "platform-team"
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    - containerPort: 443
      name: https
      protocol: TCP
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: nginx-logs
      mountPath: /var/log/nginx
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-configuration
  - name: nginx-logs
    emptyDir: {}
```

```bash
# Aplicar manifesto
kubectl apply -f nginx-pod.yaml

# Ver status do Pod
kubectl get pod nginx-production -n production

# Descrever Pod detalhadamente
kubectl describe pod nginx-production -n production

# Ver logs
kubectl logs nginx-production -n production

# Atualizar manifesto e reaplicar
kubectl apply -f nginx-pod.yaml

# Deletar Pod
kubectl delete -f nginx-pod.yaml
# ou
kubectl delete pod nginx-production -n production
```

## 3. ReplicaSets: Garantindo Disponibilidade

### 3.1. Motivação e Propósito

A natureza efêmera dos Pods, embora benéfica para flexibilidade e escalabilidade, introduz desafios significativos para disponibilidade de aplicações. Um Pod executando sozinho não oferece proteção contra falhas - se o Pod falha ou o nó que o hospeda torna-se indisponível, a aplicação fica completamente inacessível até intervenção manual. Esta limitação motivou a criação de controllers que garantem disponibilidade através de replicação automática.

ReplicaSets implementam funcionalidade fundamental de manter conjunto estável de Pods réplica executando a qualquer momento. Se um Pod gerenciado por ReplicaSet falha ou é deletado, o ReplicaSet automaticamente cria novo Pod para substituí-lo, garantindo que o número desejado de réplicas seja mantido. Esta recuperação automática constitui fundação da resiliência de aplicações no Kubernetes.

```yaml
# ReplicaSet básico
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### 3.2. Componentes de um ReplicaSet

Um ReplicaSet consiste em três componentes principais que trabalham em conjunto para manter o número desejado de réplicas:

#### 3.2.1. Replicas

O campo `replicas` especifica o número de Pods idênticos que devem estar executando. O ReplicaSet controller observa continuamente o estado atual e cria ou deleta Pods conforme necessário para alcançar este número desejado.

#### 3.2.2. Selector

O `selector` define como o ReplicaSet identifica quais Pods gerenciar. ReplicaSets utilizam set-based selectors que oferecem expressividade maior que equality-based selectors dos ReplicationControllers legados. O selector deve corresponder aos labels definidos no template do Pod.

```yaml
# Exemplos de selectors
selector:
  matchLabels:
    app: nginx
    tier: frontend

# Selector mais complexo com matchExpressions
selector:
  matchExpressions:
  - key: app
    operator: In
    values:
    - nginx
    - httpd
  - key: environment
    operator: NotIn
    values:
    - development
```

#### 3.2.3. Template

O `template` define especificação completa dos Pods que o ReplicaSet criará. Este template inclui metadata (especialmente labels) e spec dos containers. É essencial que os labels no template correspondam ao selector, caso contrário o ReplicaSet não conseguirá gerenciar os Pods que cria.

### 3.3. Funcionamento do ReplicaSet Controller

O ReplicaSet controller implementa loop de reconciliação que executa continuamente:

1. **Observar**: O controller observa o estado atual através da API do Kubernetes, determinando quantos Pods com labels correspondentes ao selector existem
2. **Comparar**: Compara o número atual de réplicas com o número desejado especificado
3. **Agir**: Se necessário, cria novos Pods (quando réplicas atuais < desejadas) ou deleta Pods existentes (quando réplicas atuais > desejadas)
4. **Repetir**: O ciclo repete continuamente, garantindo convergência ao estado desejado

```bash
# Criar ReplicaSet
kubectl apply -f replicaset.yaml

# Verificar status
kubectl get replicaset nginx-replicaset
kubectl get rs nginx-replicaset

# Ver Pods gerenciados
kubectl get pods -l app=nginx

# Descrever ReplicaSet
kubectl describe rs nginx-replicaset

# Simular falha deletando Pod
kubectl delete pod nginx-replicaset-xxxxx

# Observar ReplicaSet criando novo Pod automaticamente
kubectl get pods -l app=nginx -w
```

### 3.4. Escalabilidade com ReplicaSets

ReplicaSets facilitam escalabilidade horizontal através de ajuste simples do número de réplicas. Este escalamento pode ser realizado tanto de forma declarativa (editando o manifesto) quanto imperativa (através de comandos kubectl).

```bash
# Escalar imperativamente
kubectl scale replicaset nginx-replicaset --replicas=5

# Escalar declarativamente
# Editar manifesto alterando spec.replicas de 3 para 5
kubectl apply -f replicaset.yaml

# Verificar escalamento
kubectl get rs nginx-replicaset
kubectl get pods -l app=nginx

# Escalar para baixo
kubectl scale replicaset nginx-replicaset --replicas=2
```

A escalabilidade imperativa oferece conveniência para ajustes temporários ou experimentação, mas escalabilidade declarativa mantém manifestos sincronizados com estado do cluster, evitando divergência entre configuração documentada e realidade operacional.

### 3.5. Limitações dos ReplicaSets

Embora ReplicaSets garantam disponibilidade através de replicação, apresentam limitação crítica que os torna inadequados para uso direto em produção: não suportam atualizações controladas de versão de imagens ou mudanças na configuração de Pods.

Quando o template de Pod em um ReplicaSet é alterado (por exemplo, atualizando a versão da imagem), o ReplicaSet não atualiza Pods existentes automaticamente. Pods antigos continuam executando indefinidamente, e apenas novos Pods criados após a mudança utilizarão o template atualizado. Para forçar atualização, seria necessário deletar todos os Pods existentes, causando downtime completo da aplicação.

```yaml
# ReplicaSet original com nginx:1.20
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:1.20  # Versão antiga

---
# Tentativa de atualizar para nginx:1.21
# Esta mudança NÃO afeta Pods existentes
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:1.21  # Versão nova
```

```bash
# Aplicar manifesto atualizado
kubectl apply -f replicaset.yaml

# Pods existentes mantêm versão antiga
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# Para forçar atualização, necessário deletar Pods manualmente
kubectl delete pods -l app=nginx

# Novos Pods criados usarão nova imagem
# Mas isso causa downtime durante recriação
```

Esta limitação fundamental motivou a criação de Deployments, que gerenciam ReplicaSets automaticamente e implementam estratégias sofisticadas de atualização como rolling updates.

## 4. Deployments: Gestão de Ciclo de Vida

### 4.1. Conceito e Vantagens

Deployments constituem abstração de nível superior que gerencia ReplicaSets e fornece capacidades declarativas para atualização de Pods e ReplicaSets. Enquanto ReplicaSets focam exclusivamente em manter número desejado de réplicas, Deployments adicionam funcionalidade crítica para gestão de ciclo de vida de aplicações em produção.

As capacidades chave que Deployments adicionam incluem:

- **Rolling updates**: Atualizações graduais que substituem Pods antigos por novos de forma controlada, mantendo disponibilidade
- **Rollback**: Reversão automática ou manual a versões anteriores em caso de problemas
- **Histórico de revisões**: Manutenção de histórico de ReplicaSets anteriores, permitindo rollback a qualquer versão
- **Pausa e retomada**: Capacidade de pausar rollouts para fazer múltiplas alterações antes de aplicá-las
- **Status de progresso**: Informações detalhadas sobre progresso de rollouts e condições de erro

Deployments provêm atualizações declarativas para Pods e ReplicaSets, permitindo descrever um estado desejado e o Deployment Controller alterar o estado atual para o estado desejado a uma taxa controlada.

```yaml
# Deployment completo
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

### 4.2. Estrutura de um Deployment

A estrutura de um Deployment é muito similar à de um ReplicaSet, com adição de campos relacionados a estratégias de atualização. A seção `template` em um Deployment é idêntica ao template de Pod em ReplicaSet, e a lógica de seleção funciona da mesma forma.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application-deployment
  labels:
    app: application
    version: v1.0
spec:
  # Número desejado de réplicas
  replicas: 5
  
  # Selector que identifica Pods gerenciados
  selector:
    matchLabels:
      app: application
  
  # Template de Pod
  template:
    metadata:
      labels:
        app: application
        version: v1.0
    spec:
      containers:
      - name: application
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
  
  # Estratégia de atualização
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  
  # Tempo mínimo para considerar Pod ready
  minReadySeconds: 10
  
  # Tempo máximo para rollout antes de considerar falho
  progressDeadlineSeconds: 600
  
  # Número de ReplicaSets antigos a manter
  revisionHistoryLimit: 10
```

### 4.3. Estratégias de Atualização

Deployments suportam duas estratégias principais de atualização, especificadas no campo `strategy.type`:

#### 4.3.1. RollingUpdate

A estratégia RollingUpdate (padrão) atualiza Pods gradualmente, garantindo que aplicação permaneça disponível durante o processo. Novos Pods são criados com configuração atualizada enquanto Pods antigos são terminados de forma controlada.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    # Número máximo de Pods além de replicas durante update
    maxSurge: 1
    
    # Número máximo de Pods indisponíveis durante update
    maxUnavailable: 0
```

- **maxSurge**: Especifica quantos Pods adicionais além de `replicas` podem existir temporariamente durante rollout. Pode ser número absoluto ou percentual. Valor maior acelera rollout mas consome mais recursos temporariamente.

- **maxUnavailable**: Especifica quantos Pods podem estar indisponíveis durante rollout. Valor 0 garante zero downtime mas requer recursos suficientes para executar Pods extras (maxSurge > 0).

Exemplo de rollout com `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 1`:

1. Estado inicial: 3 Pods antigos executando
2. Criar 1 novo Pod (maxSurge permite 4 Pods totais temporariamente)
3. Aguardar novo Pod ficar ready
4. Terminar 1 Pod antigo (maxUnavailable permite 1 Pod indisponível)
5. Repetir até todos os 3 Pods serem novos

#### 4.3.2. Recreate

A estratégia Recreate termina todos os Pods antigos antes de criar novos, resultando em downtime completo mas garantindo que versões antigas e novas nunca executem simultaneamente.

```yaml
strategy:
  type: Recreate
```

Esta estratégia é apropriada para aplicações que não suportam múltiplas versões executando simultaneamente, como bancos de dados que requerem migrações de schema incompatíveis entre versões.

### 4.4. Operações com Deployments

#### 4.4.1. Criação e Atualização

```bash
# Criar Deployment
kubectl apply -f deployment.yaml

# Verificar status
kubectl get deployment nginx-deployment
kubectl get deploy nginx-deployment
```

#### 4.4.2. Rollback

A capacidade de reverter a versões anteriores constitui funcionalidade crítica para ambientes de produção, permitindo recuperação rápida de atualizações problemáticas.

```bash
# Fazer rollback para versão anterior
kubectl rollout undo deployment/nginx-deployment

# Fazer rollback para revisão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Verificar status após rollback
kubectl rollout status deployment/nginx-deployment

# Verificar histórico atualizado
kubectl rollout history deployment/nginx-deployment
```

#### 4.4.3. Pausa e Retomada

Pausar um Deployment permite fazer múltiplas alterações antes de acionar novo rollout, útil quando mudanças relacionadas devem ser aplicadas atomicamente.

```bash
# Pausar Deployment
kubectl rollout pause deployment/nginx-deployment

# Fazer múltiplas alterações
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=500m,memory=512Mi

# Retomar rollout
kubectl rollout resume deployment/nginx-deployment

# Verificar status
kubectl rollout status deployment/nginx-deployment
```

#### 4.4.4. Escalabilidade

```bash
# Escalar imperativo
kubectl scale deployment nginx-deployment --replicas=10

# Escalar declarativo
# Editar deployment.yaml alterando spec.replicas
kubectl apply -f deployment.yaml

# Verificar escalamento
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

### 4.5. Relação entre Deployment, ReplicaSet e Pods

Deployments criam e gerenciam ReplicaSets automaticamente. Cada vez que o template de Pod em um Deployment é alterado, um novo ReplicaSet é criado. O Deployment então escala gradualmente o novo ReplicaSet para cima enquanto escala o antigo para baixo, implementando rolling update.

```bash
# Observar relação hierárquica
kubectl get deployment nginx-deployment
kubectl get replicaset -l app=nginx
kubectl get pods -l app=nginx

# Ver owner references (quem criou cada recurso)
kubectl get pods -l app=nginx -o yaml | grep -A 5 ownerReferences
```

Cada ReplicaSet mantido no histórico pode ser usado para rollback. O campo `revisionHistoryLimit` controla quantos ReplicaSets antigos são mantidos (padrão 10).

```yaml
spec:
  revisionHistoryLimit: 10  # Manter 10 revisões antigas
```

### 4.6. Boas Práticas com Deployments

#### 4.6.1. Versionamento de Imagens

Utilizar tags específicas de versão ao invés de `latest` constitui prática essencial. Tags como `latest` ou `stable` mudam ao longo do tempo, tornando impossível determinar qual versão específica está executando ou reverter a versão anterior.

```yaml
# Evitar
containers:
- name: application
  image: myapp:latest

# Preferir
containers:
- name: application
  image: myapp:v1.2.3  # Tag específica, idealmente commit hash
```

#### 4.6.2. Health Checks

Definir liveness e readiness probes apropriadas garante que Deployments considerem Pods ready apenas quando realmente capazes de processar tráfego, e reiniciem automaticamente Pods em estados problemáticos.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
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

#### 4.6.3. Resource Requests e Limits

Especificar requests e limits apropriados previne que containers monopolizem recursos e permite scheduler tomar decisões informadas de placement.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

#### 4.6.4. Configuração de Rolling Update

Ajustar `maxSurge` e `maxUnavailable` conforme requisitos específicos de disponibilidade e capacidade:

```yaml
# Zero downtime, requer recursos extras
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Rollout mais rápido, aceita indisponibilidade temporária
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1
```

## 5. Services: Abstraindo Acesso a Pods

### 5.1. Necessidade de Services

Pods são efêmeros e podem ser criados, destruídos e substituídos frequentemente. Cada Pod recebe endereço IP único, mas este IP muda toda vez que o Pod é recriado. Adicionalmente, Deployments criam e gerenciam múltiplas réplicas de Pods, cada uma com seu próprio IP. Aplicações cliente necessitam forma estável de comunicar com estas aplicações backend sem rastrear manualmente IPs individuais de Pods.

Services resolvem este problema fornecendo abstração estável sobre conjunto dinâmico de Pods. Um Service recebe endereço IP virtual estável (ClusterIP) e nome DNS que permanecem constantes independentemente de mudanças nos Pods backend. O kube-proxy em cada nó mantém regras de rede que encaminham tráfego destinado ao Service para Pods backend apropriados, implementando balanceamento de carga automático.

### 5.2. Tipos de Services

Kubernetes suporta quatro tipos principais de Services, cada um apropriado para diferentes cenários de exposição:

#### 5.2.1. ClusterIP

ClusterIP (padrão) expõe o Service em IP interno ao cluster. O Service torna-se acessível apenas de dentro do cluster, apropriado para comunicação entre microsserviços.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80          # Porta do Service
    targetPort: 8080  # Porta do container
```

#### 5.2.2. NodePort

NodePort expõe o Service na mesma porta em todos os nós do cluster. Clientes externos podem acessar o Service através de `<NodeIP>:<NodePort>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080  # Porta nos nós (30000-32767)
```

#### 5.2.3. LoadBalancer

LoadBalancer provê o Service através de load balancer de provedor cloud, criando automaticamente NodePort e ClusterIP subjacentes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

#### 5.2.4. ExternalName

ExternalName mapeia o Service a nome DNS externo, útil para referenciar serviços externos ao cluster através de DNS interno.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.external.com
```

### 5.3. Seletores e Endpoints

Services utilizam label selectors para determinar quais Pods devem receber tráfego. Quando um Service é criado, o Endpoints Controller automaticamente cria objeto Endpoints contendo IPs de todos os Pods que correspondem ao selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: application-service
spec:
  selector:
    app: application
    tier: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

```bash
# Ver Service
kubectl get service application-service

# Ver Endpoints associados
kubectl get endpoints application-service

# Descrever Service mostra Endpoints
kubectl describe service application-service
```

O objeto Endpoints é atualizado automaticamente quando Pods são criados ou destruídos, garantindo que o Service sempre encaminhe tráfego apenas para Pods ready.

### 5.4. Descoberta de Serviços via DNS

Kubernetes executa servidor DNS interno (tipicamente CoreDNS) que automaticamente cria registros DNS para Services. Pods são configurados automaticamente para utilizar este DNS, permitindo descoberta de serviços através de nomes.

Formato dos nomes DNS:
- `<service-name>.<namespace>.svc.cluster.local` (nome completo)
- `<service-name>.<namespace>` (pode omitir domínio cluster)
- `<service-name>` (para Services no mesmo namespace)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
spec:
  containers:
  - name: client
    image: busybox
    command: ['sh', '-c', 'while true; do wget -O- http://backend-service.production; sleep 5; done']
```

```bash
# Testar resolução DNS de dentro do cluster
kubectl run test-pod --image=busybox -it --rm -- nslookup backend-service.production

# Testar conectividade
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://backend-service.production
```

### 5.5. Criação de Services

#### 5.5.1. Criação Declarativa

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: production
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
  sessionAffinity: ClientIP  # Opcional: afinidade de sessão
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

```bash
# Aplicar manifesto
kubectl apply -f service.yaml

# Verificar Service
kubectl get service nginx-service -n production

# Ver detalhes
kubectl describe service nginx-service -n production

# Testar Service de dentro do cluster
kubectl run test --image=busybox -it --rm -n production -- wget -O- http://nginx-service
```

#### 5.5.2. Criação Imperativa

```bash
# Expor Deployment
kubectl expose deployment nginx-deployment --port=80 --target-port=80 --type=ClusterIP

# Expor Pod
kubectl expose pod nginx-pod --port=80 --target-port=80

# Criar Service com NodePort
kubectl expose deployment nginx-deployment --port=80 --target-port=80 --type=NodePort

# Gerar manifesto sem criar
kubectl expose deployment nginx-deployment --port=80 --type=ClusterIP --dry-run=client -o yaml > service.yaml
```

### 5.6. Relação entre Services e Deployments

Services e Deployments são independentes mas complementares. Um Deployment gerencia Pods enquanto um Service fornece acesso estável a estes Pods. Tipicamente existe relação um-para-um entre Services e Deployments em arquiteturas simples, mas cenários mais complexos podem envolver múltiplos Services apontando para o mesmo conjunto de Pods (diferenciando por portas) ou um Service balanceando entre Pods de múltiplos Deployments.

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80

---
# Service correspondente
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
    tier: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

A chave para conectar Services e Pods são os labels. O selector do Service deve corresponder aos labels dos Pods criados pelo Deployment.

### 5.7. Gestão Declarativa versus Imperativa

A gestão declarativa através de manifestos YAML versionados representa prática recomendada para Services em produção, assim como para outros recursos. Manifestos servem como documentação, podem ser revisados através de pull requests, e mantêm histórico completo de mudanças.

```bash
# Fluxo declarativo completo

# 1. Criar diretório de manifestos
mkdir -p k8s/production

# 2. Criar manifestos
cat > k8s/production/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
  namespace: production
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
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
EOF

cat > k8s/production/service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: application-service
  namespace: production
spec:
  selector:
    app: application
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOF

# 3. Aplicar todos os manifestos
kubectl apply -f k8s/production/

# 4. Verificar recursos criados
kubectl get all -n production

# 5. Fazer alterações nos manifestos
vim k8s/production/deployment.yaml  # Alterar replicas ou image

# 6. Reaplicar
kubectl apply -f k8s/production/

# 7. Verificar mudanças
kubectl rollout status deployment/application -n production
```

### 5.8. Debugging e Troubleshooting de Services

```bash
# Verificar se Service existe
kubectl get service application-service

# Ver Endpoints do Service
kubectl get endpoints application-service
kubectl describe endpoints application-service

# Se Endpoints estiver vazio, verificar:
# 1. Pods existem com labels corretos?
kubectl get pods --show-labels

# 2. Labels dos Pods correspondem ao selector do Service?
kubectl get service application-service -o jsonpath='{.spec.selector}'

# 3. Pods estão ready?
kubectl get pods

# Testar conectividade de dentro do cluster
kubectl run debug-pod --image=busybox -it --rm -- wget -O- http://application-service

# Ver logs do kube-proxy para troubleshooting de rede
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Verificar regras iptables (se usando modo iptables)
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl exec -n kube-system kube-proxy-xxxxx -- iptables-save | grep application-service
```

## 6. Organizando Recursos com Namespaces

### 6.1. Propósito e Casos de Uso

Namespaces fornecem mecanismo para dividir recursos de cluster entre múltiplos usuários, equipes ou ambientes. Cada namespace provê escopo isolado para nomes, permitindo que recursos com mesmo nome existam em namespaces diferentes.

Casos de uso comuns incluem:

- **Separação de ambientes**: development, staging, production em mesmo cluster
- **Isolamento de equipes**: cada equipe opera em namespace próprio
- **Multi-tenancy**: múltiplos clientes compartilhando cluster
- **Separação de projetos**: diferentes projetos ou aplicações

```bash
# Listar namespaces
kubectl get namespaces
kubectl get ns

# Criar namespace
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Deletar namespace (deleta todos os recursos)
kubectl delete namespace development
```

### 6.2. Criação Declarativa de Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
  annotations:
    description: "Production environment for all services"
```

```bash
# Aplicar manifesto
kubectl apply -f namespace.yaml

# Verificar namespace criado
kubectl get namespace production
kubectl describe namespace production
```

### 6.3. Resource Quotas e Limit Ranges

Namespaces permitem aplicação de políticas de governança através de ResourceQuotas e LimitRanges.

#### 6.3.1. ResourceQuota

ResourceQuotas limitam consumo agregado de recursos em namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "10"
    pods: "50"
```

#### 6.3.2. LimitRange

LimitRanges definem limites padrão e máximos para containers individuais.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limitrange
  namespace: production
spec:
  limits:
  - max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    type: Container
```

### 6.4. Trabalhando com Namespaces

```bash
# Criar recursos em namespace específico
kubectl apply -f deployment.yaml -n production

# Definir namespace padrão para contexto
kubectl config set-context --current --namespace=production

# Verificar namespace padrão
kubectl config view --minify | grep namespace:

# Listar recursos em namespace
kubectl get all -n production
kubectl get pods -n production
kubectl get services -n production

# Listar recursos em todos os namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Deletar recurso em namespace
kubectl delete pod nginx-pod -n production

# Deletar todos os recursos em namespace (mantém namespace)
kubectl delete all --all -n production
```

### 6.5. Comunicação Entre Namespaces

Services em um namespace são acessíveis de outros namespaces através de nome DNS completo.

```yaml
# Service no namespace "backend"
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: backend
spec:
  selector:
    app: database
  ports:
  - port: 5432

---
# Pod no namespace "frontend" acessando Service
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: frontend
spec:
  containers:
  - name: web
    image: webapp:latest
    env:
    - name: DATABASE_URL
      value: "postgresql://database-service.backend.svc.cluster.local:5432/appdb"
```

NetworkPolicies podem restringir comunicação entre namespaces quando necessário para segurança.

## 6. Conclusões

A orquestração efetiva de containers no Kubernetes requer compreensão profunda das abstrações hierárquicas que a plataforma oferece, desde Pods como unidade fundamental até Deployments como mecanismo sofisticado de gestão de ciclo de vida. Cada nível de abstração adiciona capacidades específicas que endereçam desafios operacionais distintos, formando sistema coeso que facilita gestão de aplicações distribuídas complexas.

A evolução de Pods simples através de ReplicaSets até Deployments demonstra como o Kubernetes progressivamente adiciona funcionalidades de resiliência, disponibilidade e gestão de versões. Pods fornecem encapsulamento básico e co-localização de containers, ReplicaSets garantem disponibilidade através de replicação automática, e Deployments completam a stack com capacidades essenciais de rolling updates e rollbacks. Esta organização em camadas permite que desenvolvedores utilizem abstrações apropriadas ao nível de controle necessário, desde Pods diretos para casos especiais até Deployments para aplicações stateless típicas.

Services complementam esta arquitetura fornecendo camada de rede que abstrai a natureza efêmera dos Pods, oferecendo endpoints estáveis com descoberta automática através de DNS e balanceamento de carga integrado. A separação clara entre gestão de ciclo de vida (Deployments) e exposição de rede (Services) exemplifica princípios de design do Kubernetes baseados em composição de componentes especializados. Esta separação facilita evolução independente de aspectos diferentes da arquitetura e permite reutilização de Services entre múltiplos backends.

A transição de abordagens imperativas para declarativas representa mudança fundamental de paradigma que, embora inicialmente possa parecer mais complexa, oferece benefícios substanciais para operações em escala. Manifestos YAML versionados em sistemas de controle de versão fornecem documentação executável, histórico completo de mudanças, capacidade de revisão colaborativa, e fundação para automação através de pipelines de CI/CD. A gestão declarativa transforma infraestrutura em código, aplicando ao domínio operacional as mesmas práticas de engenharia que há décadas beneficiam desenvolvimento de software.

Namespaces completam o modelo organizacional do Kubernetes, permitindo isolamento lógico e aplicação de políticas de governança em clusters compartilhados. A capacidade de particionar clusters em ambientes virtuais distintos, cada um com suas próprias quotas de recursos e políticas de acesso, democratiza acesso a infraestrutura Kubernetes enquanto mantém controles necessários para operação segura e eficiente. Este modelo multi-tenant reduz custos operacionais através de consolidação enquanto preserva isolamento adequado entre cargas de trabalho.

## 7. Referências Bibliográficas

### 7.1. Documentação Oficial do Kubernetes

- Kubernetes Documentation. "Pods". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/pods/

- Kubernetes Documentation. "ReplicaSet". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

- Kubernetes Documentation. "Deployments". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

- Kubernetes Documentation. "Services". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/services-networking/service/

- Kubernetes Documentation. "Namespaces". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/

### 7.2. Conceitos de Workloads

- Kubernetes Documentation. "Workload Resources". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/

- Kubernetes Documentation. "Managing Resources". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

- Kubernetes Documentation. "Configure Liveness, Readiness and Startup Probes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### 7.3. Rede e Services

- Kubernetes Documentation. "Service Networking". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/services-networking/

- Kubernetes Documentation. "DNS for Services and Pods". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

- Kubernetes Documentation. "Connecting Applications with Services". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tutorials/services/connect-applications-service/

### 7.4. Governança e Organização

- Kubernetes Documentation. "Resource Quotas". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/policy/resource-quotas/

- Kubernetes Documentation. "Limit Ranges". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/policy/limit-range/

- Kubernetes Documentation. "Configure Default Memory Requests and Limits for a Namespace". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/

### 7.5. Best Practices

- Kubernetes Documentation. "Configuration Best Practices". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/configuration/overview/

- Kubernetes Documentation. "Running in Production". Kubernetes.io. Disponível em: https://kubernetes.io/docs/setup/best-practices/

## 8. Apêndices

### Apêndice A: Estrutura de Manifesto Completo

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
  namespace: production
  labels:
    app: web
    version: v1.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1.0
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http
```

### Apêndice B: Comandos kubectl Essenciais para Workloads

```bash
# Pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -f
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- /bin/bash
kubectl delete pod <pod-name>

# Deployments
kubectl get deployments
kubectl get deploy
kubectl describe deployment <deployment-name>
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
kubectl rollout undo deployment/<deployment-name>
kubectl scale deployment <deployment-name> --replicas=5
kubectl set image deployment/<deployment-name> <container>=<image>
kubectl delete deployment <deployment-name>

# ReplicaSets
kubectl get replicasets
kubectl get rs
kubectl describe rs <replicaset-name>
kubectl scale rs <replicaset-name> --replicas=3
kubectl delete rs <replicaset-name>

# Services
kubectl get services
kubectl get svc
kubectl describe service <service-name>
kubectl get endpoints <service-name>
kubectl delete service <service-name>

# Namespaces
kubectl get namespaces
kubectl get ns
kubectl create namespace <name>
kubectl delete namespace <name>
kubectl config set-context --current --namespace=<name>

# Troubleshooting
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes
kubectl top pods
kubectl port-forward <pod-name> 8080:80
```

### Apêndice C: Padrões de Labels Recomendados

```yaml
metadata:
  labels:
    # Labels recomendadas pela Kubernetes
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: kubectl
    
    # Labels personalizadas
    environment: production
    team: platform
    cost-center: engineering
```

### Apêndice D: Troubleshooting Comum

#### D.1. Pods em CrashLoopBackOff

```bash
# Ver logs do container
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Descrever pod
kubectl describe pod <pod-name>

# Verificar eventos
kubectl get events --field-selector involvedObject.name=<pod-name>

# Verificar resource limits
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'
```

#### D.2. Service Sem Endpoints

```bash
# Verificar endpoints
kubectl get endpoints <service-name>

# Verificar selector do service
kubectl get service <service-name> -o jsonpath='{.spec.selector}'

# Verificar labels dos pods
kubectl get pods --show-labels

# Verificar se selector corresponde aos labels
kubectl get pods -l app=myapp
```

#### D.3. Deployment Não Atualiza Pods

```bash
# Verificar status do rollout
kubectl rollout status deployment/<name>

# Ver histórico
kubectl rollout history deployment/<name>

# Ver eventos
kubectl describe deployment <name>

# Forçar recriação de pods
kubectl rollout restart deployment/<name>
```

### Apêndice E: Glossário e Termos Técnicos

**ClusterIP**: Tipo padrão de Service que expõe aplicação em IP interno ao cluster, acessível apenas de dentro do cluster.

**Controller**: Componente que implementa loop de controle, observando estado do cluster e tomando ações para convergir estado atual ao desejado.

**Declarative Configuration**: Abordagem onde administradores especificam estado desejado em manifestos e o sistema trabalha para alcançá-lo.

**Deployment**: Recurso que fornece atualizações declarativas para Pods e ReplicaSets, gerenciando rollouts, rollbacks e histórico de versões.

**Downtime**: Período durante o qual aplicação ou serviço está indisponível para usuários.

**Endpoints**: Objeto que rastreia endereços IP e portas de Pods que implementam um Service, atualizado automaticamente pelo Endpoints Controller.

**Ephemeral**: Característica de recursos transitórios e descartáveis, que podem ser criados e destruídos frequentemente. Pods são efêmeros por design.

**Imperative Configuration**: Abordagem onde administradores especificam comandos diretos para executar ações específicas no cluster.

**Label**: Par chave-valor anexado a objetos para identificação e seleção. Labels não fornecem unicidade mas permitem organização flexível.

**Label Selector**: Mecanismo para filtrar e selecionar recursos baseado em labels, utilizado por Controllers e Services para identificar Pods gerenciados.

**LimitRange**: Política que define valores padrão, mínimos e máximos para resource requests e limits de containers em namespace.

**Liveness Probe**: Verificação periódica para determinar se container está executando corretamente. Falhas resultam em reinicialização do container.

**Manifest**: Arquivo YAML ou JSON que descreve declarativamente recursos do Kubernetes, incluindo toda sua configuração.

**maxSurge**: Parâmetro de rolling update que especifica número máximo de Pods além de replicas que podem existir temporariamente durante atualização.

**maxUnavailable**: Parâmetro de rolling update que especifica número máximo de Pods que podem estar indisponíveis durante atualização.

**Namespace**: Mecanismo de isolamento lógico que particiona cluster em múltiplos clusters virtuais, fornecendo escopo para nomes de recursos.

**NodePort**: Tipo de Service que expõe aplicação em porta específica em todos os nós do cluster, permitindo acesso externo.

**Pod**: Menor unidade deployável no Kubernetes, consistindo em um ou mais containers que compartilham recursos de rede e armazenamento.

**Port Forwarding**: Túnel que encaminha tráfego de porta local para porta em Pod, útil para debugging e acesso direto a aplicações.

**Readiness Probe**: Verificação que determina se container está pronto para receber tráfego. Falhas removem Pod dos endpoints do Service.

**Reconciliation Loop**: Processo contínuo onde controllers comparam estado atual com desejado e executam ações para convergência.

**Recreate Strategy**: Estratégia de atualização de Deployment que termina todos os Pods antigos antes de criar novos, causando downtime.

**Replica**: Instância individual de Pod gerenciado por controller como ReplicaSet ou Deployment.

**ReplicaSet**: Controller que mantém conjunto estável de Pods réplica executando a qualquer momento, garantindo disponibilidade através de replicação automática.

**Resource Limits**: Quantidade máxima de recursos (CPU, memória) que container pode consumar, enforçada pela container runtime.

**Resource Quota**: Política que limita consumo agregado de recursos em namespace, prevenindo monopolização de recursos do cluster.

**Resource Requests**: Quantidade de recursos que container declara necessitar, utilizada pelo scheduler para decisões de placement.

**Rollback**: Reversão de Deployment a versão anterior, revertendo atualizações problemáticas.

**Rolling Update**: Estratégia de atualização que substitui Pods antigos por novos gradualmente, mantendo disponibilidade durante processo.

**Rollout**: Processo de atualização de Deployment, criando novo ReplicaSet e migrando tráfego progressivamente.

**Selector**: Mecanismo para identificar conjunto de recursos baseado em labels, utilizado por Controllers e Services.

**Service**: Abstração que define política de acesso lógico a conjunto de Pods, fornecendo endpoint estável com descoberta DNS e balanceamento de carga.

**Service Discovery**: Mecanismo que permite aplicações descobrir e comunicar com outros serviços, implementado no Kubernetes através de DNS.

**SessionAffinity**: Configuração de Service que garante que requisições do mesmo cliente sejam direcionadas ao mesmo Pod backend.

**Stateless Application**: Aplicação que não mantém estado local entre requisições, facilitando escalabilidade horizontal e recuperação de falhas.

**Strategy**: Configuração que define como Deployments executam atualizações, incluindo RollingUpdate e Recreate.

**Template**: Especificação de Pod dentro de ReplicaSet ou Deployment, definindo configuração completa dos Pods que serão criados.

**TargetPort**: Porta em container para qual Service encaminha tráfego, especificada em definição de Service.

**Zero Downtime Deployment**: Estratégia de atualização que mantém aplicação disponível durante todo processo de deployment, tipicamente implementada através de rolling updates. eventos relacionados ao Pod
kubectl describe pod nginx-pod

