<!-- markdownlint-disable -->
# Bloco D - Horizontal Pod Autoscaler no Kubernetes

## Resumo Executivo

O Horizontal Pod Autoscaler (HPA) representa um dos mecanismos fundamentais de escalabilidade automática no ecossistema Kubernetes, permitindo que aplicações containerizadas ajustem dinamicamente seu número de réplicas em resposta a variações de carga e utilização de recursos. Este documento apresenta uma análise técnica abrangente sobre estratégias de escalabilidade em ambientes Kubernetes, com ênfase particular no HPA e seus componentes associados.

A escalabilidade horizontal, ao contrário da vertical, distribui a carga de trabalho entre múltiplas instâncias de aplicação, proporcionando maior resiliência, disponibilidade e eficiência no uso de recursos computacionais. O HPA opera em conjunto com o Metrics Server, componente responsável pela coleta e exposição de métricas de recursos, formando assim a base da pipeline de autoescalamento nativa do Kubernetes. Através de configurações declarativas expressas em manifestos YAML, administradores podem definir políticas sofisticadas de escalamento que respondem automaticamente a diferentes cenários operacionais, desde picos súbitos de tráfego até períodos de baixa utilização.

Este documento explora os fundamentos conceituais da escalabilidade, diferencia as abordagens vertical e horizontal, detalha a arquitetura e funcionamento do Metrics Server, apresenta as versões v1 e v2 do HPA com suas respectivas capacidades, e demonstra através de exemplos práticos como implementar e testar estratégias de autoescalamento. Adicionalmente, são discutidas técnicas avançadas como janelas de estabilização, políticas de scale-up e scale-down, além de métricas customizadas que ampliam significativamente as possibilidades de escalamento além dos recursos básicos de CPU e memória.

## 1. Introdução e Conceitos

### 1.1. Contexto de Escalabilidade em Ambientes Distribuídos

A escalabilidade constitui um dos pilares fundamentais da arquitetura de microsserviços e aplicações cloud-native. Em ambientes de produção modernos, aplicações enfrentam variações significativas e frequentemente imprevisíveis em suas cargas de trabalho. Estas variações podem originar-se de múltiplos fatores, incluindo padrões de acesso de usuários ao longo do dia, eventos sazonais, campanhas de marketing, ou mesmo situações adversas como ataques DDoS.

A capacidade de adaptar dinamicamente os recursos computacionais alocados a uma aplicação não apenas garante que os usuários finais experimentem tempos de resposta aceitáveis durante períodos de alta demanda, mas também otimiza custos operacionais ao evitar o provisionamento excessivo de infraestrutura durante momentos de baixa utilização. Tradicionalmente, o provisionamento de capacidade seguia abordagens estáticas, onde a infraestrutura era dimensionada para suportar os picos máximos esperados, resultando em desperdício significativo de recursos durante a maior parte do tempo operacional.

O Kubernetes, como plataforma de orquestração de containers, provê mecanismos sofisticados que permitem implementar estratégias de escalabilidade automática, eliminando ou minimizando a necessidade de intervenção manual no processo de ajuste de capacidade. Estes mecanismos fundamentam-se em princípios declarativos, onde administradores especificam o estado desejado do sistema, cabendo ao orquestrador garantir a convergência entre o estado atual e o desejado.

### 1.2. Resiliência e Tolerância a Falhas

A resiliência de aplicações em clusters Kubernetes está intrinsecamente relacionada à sua capacidade de escalabilidade. Aplicações que podem escalar horizontalmente distribuem automaticamente a carga entre múltiplas instâncias, garantindo que a falha de uma ou mais réplicas não comprometa completamente a disponibilidade do serviço. Esta característica é especialmente crítica em arquiteturas de microsserviços, onde a indisponibilidade de um componente pode afetar cascatas de serviços dependentes.

O conceito de autoescala complementa a resiliência ao permitir que o sistema reaja proativamente a condições adversas. Por exemplo, se uma aplicação começa a experimentar latências elevadas devido ao aumento de carga, o mecanismo de autoescalamento pode provisionar réplicas adicionais antes que ocorram timeouts ou falhas nas requisições. Esta capacidade de resposta automática reduz significativamente o Mean Time To Recovery (MTTR) e melhora métricas gerais de disponibilidade.

### 1.3. Testes de Carga e Validação de Comportamento

A implementação eficaz de estratégias de autoescalamento requer compreensão profunda do comportamento da aplicação sob diferentes condições de carga. Testes de carga e estresse são práticas essenciais para validar configurações de escalamento, identificar gargalos de performance, e estabelecer baselines de utilização de recursos.

Ferramentas especializadas como FortIO, Apache JMeter, Locust, ou K6 permitem simular cenários realistas de tráfego, gerando cargas sintéticas que exercitam diferentes aspectos da aplicação. Estes testes devem ser conduzidos de forma sistemática, variando parâmetros como taxa de requisições, concorrência, duração dos testes, e padrões de acesso. A análise dos resultados obtidos permite calibrar adequadamente os triggers de escalamento, garantindo que o sistema responda apropriadamente às demandas reais de produção.

## 2. Tipos de Escalabilidade

### 2.1. Escala Vertical (Vertical Scaling)

A escala vertical, também conhecida como "scaling up" ou "scaling down", consiste em aumentar ou diminuir os recursos computacionais alocados a uma instância específica de aplicação. Em termos práticos, isto significa incrementar a quantidade de CPU, memória RAM, ou capacidade de armazenamento disponível para um Pod individual no Kubernetes.

#### 2.1.1. Características e Aplicabilidade

A escalabilidade vertical apresenta vantagens específicas em determinados cenários. Aplicações que mantêm estado local extensivo, utilizam estruturas de dados em memória compartilhadas, ou dependem fortemente de comunicação inter-thread beneficiam-se desta abordagem. Adicionalmente, aplicações legadas que não foram projetadas para execução distribuída frequentemente requerem escala vertical como única alternativa viável.

No contexto do Kubernetes, a escala vertical pode ser implementada manualmente através da modificação das especificações de recursos nos manifestos de Deployment, ou automaticamente utilizando o Vertical Pod Autoscaler (VPA). O VPA monitora a utilização de recursos dos Pods e ajusta automaticamente os requests e limits de CPU e memória, garantindo que os containers recebam recursos apropriados às suas necessidades operacionais.

#### 2.1.2. Limitações e Considerações

A escala vertical enfrenta limitações significativas que restringem sua aplicabilidade em ambientes de grande escala. Primeiramente, existe um limite físico para a quantidade de recursos que podem ser alocados a uma única instância, determinado pelas capacidades do hardware subjacente. Adicionalmente, o processo de escala vertical frequentemente requer reinicialização dos Pods, resultando em períodos de indisponibilidade que podem impactar negativamente a experiência dos usuários.

A ausência de redundância inerente à escala vertical representa outra preocupação crítica. Concentrar toda a capacidade computacional em um número reduzido de instâncias potentes cria pontos únicos de falha, onde a indisponibilidade de uma máquina pode comprometer significativamente a disponibilidade do serviço. Em ambientes de produção que demandam alta disponibilidade, esta característica constitui uma desvantagem substancial.

O fenômeno de ociosidade de recursos também merece atenção. Após períodos de alta demanda que motivaram escalamento vertical, a redução da carga pode resultar em recursos substancialmente subutilizados, gerando custos operacionais desnecessários. O processo de downsize, embora possível, compartilha as mesmas limitações operacionais do upsize, incluindo necessidade de reinicialização.

#### 2.1.3. Vertical Pod Autoscaler no Kubernetes

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: application-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application-deployment
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: application-container
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
      controlledResources:
      - cpu
      - memory
```

### 2.2. Escala Horizontal (Horizontal Scaling)

A escala horizontal, ou "scaling out" e "scaling in", consiste em aumentar ou diminuir o número de instâncias que executam uma aplicação. No Kubernetes, isto se traduz em adicionar ou remover réplicas de Pods que executam os mesmos containers, distribuindo a carga de trabalho entre múltiplas instâncias idênticas.

#### 2.2.1. Princípios e Vantagens

A escalabilidade horizontal fundamenta-se no princípio de distribuição de carga e redundância. Ao replicar a aplicação através de múltiplas instâncias, o sistema adquire capacidade de processar um volume maior de requisições simultaneamente, enquanto simultaneamente melhora características de resiliência e tolerância a falhas.

Esta abordagem oferece vantagens significativas em relação à escala vertical. A eliminação de limites rígidos de capacidade permite que o sistema escale teoricamente de forma ilimitada, restrito apenas pela capacidade total do cluster. A granularidade do escalamento é superior, possibilitando ajustes incrementais precisos em resposta a variações graduais de carga. O processo de adição ou remoção de réplicas geralmente ocorre sem interrupção do serviço, mantendo outras instâncias operacionais durante a transição.

#### 2.2.2. Distribuição de Carga e Balanceamento

A eficácia da escala horizontal depende fundamentalmente de mecanismos apropriados de distribuição de carga. No Kubernetes, os Services implementam balanceamento de carga nativo entre os Pods que compõem um Deployment, utilizando algoritmos como round-robin para distribuir requisições de forma equitativa. Ingress Controllers podem implementar estratégias mais sofisticadas, incluindo roteamento baseado em sessão, distribuição ponderada, ou afinidade de cliente.

A distribuição apropriada de carga garante que todas as réplicas contribuam de forma equilibrada para o processamento das requisições, evitando situações onde algumas instâncias permanecem ociosas enquanto outras experimentam sobrecarga. Métricas de utilização coletadas pelo Metrics Server refletem a média entre todas as réplicas, fornecendo visibilidade sobre a efetividade da distribuição de carga.

#### 2.2.3. Considerações sobre Estado e Consistência

Aplicações stateless adaptam-se naturalmente à escala horizontal, pois não mantêm informações de sessão ou estado que precisem ser compartilhadas entre requisições. Cada instância pode processar qualquer requisição de forma independente, simplificando significativamente a arquitetura de escalamento.

Aplicações stateful, por outro lado, apresentam desafios adicionais. O compartilhamento de estado entre múltiplas instâncias requer mecanismos de sincronização, geralmente implementados através de sistemas de cache distribuído como Redis ou Memcached, ou bancos de dados que suportam concorrência apropriada. A consistência de dados torna-se uma preocupação crítica, especialmente em cenários onde múltiplas instâncias podem tentar modificar simultaneamente o mesmo registro.

StatefulSets no Kubernetes fornecem primitivas especializadas para aplicações stateful, incluindo identidades de rede estáveis, armazenamento persistente por réplica, e ordem garantida de inicialização e terminação. No entanto, mesmo com estas facilidades, a implementação de escalamento horizontal para aplicações stateful requer design cuidadoso e compreensão profunda dos trade-offs envolvidos.

#### 2.2.4. Autoescala Baseada em Eventos

Além de métricas tradicionais de utilização de recursos, a escala horizontal pode ser acionada por eventos específicos ou métricas customizadas. O KEDA (Kubernetes Event-Driven Autoscaling) estende as capacidades do HPA nativo, permitindo escalamento baseado em eventos provenientes de diversas fontes, incluindo filas de mensagens (RabbitMQ, Kafka, Azure Service Bus), sistemas de monitoramento (Prometheus), bases de dados, e dezenas de outros event sources.

Esta abordagem event-driven é particularmente apropriada para arquiteturas de processamento assíncrono, onde o tamanho de filas de mensagens indica mais precisamente a necessidade de capacidade adicional do que métricas genéricas de CPU. Por exemplo, um sistema de processamento de pedidos pode escalar baseado no comprimento da fila de pedidos pendentes, garantindo tempos de processamento aceitáveis independentemente do volume de CPU ou memória utilizado.

## 3. Metrics Server: Fundamentos e Arquitetura

### 3.1. Visão Geral e Propósito

O Metrics Server constitui um componente crítico da arquitetura de monitoramento do Kubernetes, especificamente projetado para coletar e expor métricas de recursos de forma eficiente e escalável. Diferentemente de soluções de monitoramento completas como Prometheus ou Datadog, o Metrics Server foca exclusivamente em métricas de recursos fundamentais necessárias para pipelines de autoescalamento.

O posicionamento do Metrics Server na arquitetura do Kubernetes reflete sua função especializada. Ele implementa a Metrics API, uma API agregada que se integra nativamente ao API Server do Kubernetes, permitindo que componentes como HPA e VPA consultem métricas através da mesma interface utilizada para outros recursos do cluster. Esta integração simplifica significativamente a arquitetura de autoescalamento, eliminando dependências em sistemas externos de monitoramento para funcionalidade básica.

### 3.2. Arquitetura e Fluxo de Coleta de Métricas

O funcionamento do Metrics Server fundamenta-se em um pipeline de coleta e agregação que opera continuamente no cluster. O processo inicia-se com o Kubelet, componente que executa em cada nó do cluster e é responsável pela gestão dos containers. O Kubelet expõe métricas através de sua API, incluindo informações sobre utilização de CPU e memória tanto no nível de nó quanto no nível de Pod e container.

O Metrics Server periodicamente consulta o endpoint de métricas de cada Kubelet, coletando dados atualizados sobre utilização de recursos. A frequência padrão de coleta é de 15 segundos, balanceando a necessidade de métricas atualizadas com o overhead de comunicação de rede e processamento. Esta janela temporal implica que decisões de escalamento baseiam-se em dados com latência máxima de 15 segundos, suficiente para a maioria dos cenários operacionais.

Após coletar as métricas, o Metrics Server realiza agregação e cálculos necessários, expondo os resultados através da Metrics API. Esta API fornece dois principais recursos: métricas de nós (node metrics) e métricas de Pods (pod metrics). As métricas são expressas utilizando notação de quantidades do Kubernetes, onde valores podem ser representados em unidades como millicores (m) para CPU ou bytes com sufixos (Mi, Gi) para memória.

```bash
# Consultar métricas de nós
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} kubectl top node {}

# Consultar métricas de Pods
kubectl top pods --all-namespaces

# Verificar disponibilidade da Metrics API
kubectl get apiservices | grep metrics
```

### 3.3. Instalação e Configuração

A instalação do Metrics Server pode ser realizada através de múltiplas abordagens, cada uma apropriada para diferentes cenários e requisitos operacionais. O método mais direto consiste na aplicação de manifestos YAML oficiais mantidos pelo projeto.

```bash
# Instalação básica usando manifestos oficiais
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Instalação em alta disponibilidade (HA) para clusters de produção
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
```

Ambientes de desenvolvimento ou clusters que não utilizam certificados assinados por autoridades reconhecidas frequentemente requerem configurações adicionais. A validação de certificados TLS pode ser desabilitada através do argumento `--kubelet-insecure-tls`, embora esta prática seja inadequada para ambientes de produção devido às implicações de segurança.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      containers:
      - name: metrics-server
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.0
        args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        # Apenas para desenvolvimento - NUNCA em produção
        # - --kubelet-insecure-tls
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 1000m
            memory: 1000Mi
        ports:
        - containerPort: 10250
          name: https
          protocol: TCP
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      volumes:
      - name: tmp-dir
        emptyDir: {}
```

### 3.4. Requisitos de Rede e Segurança

O Metrics Server possui requisitos específicos de comunicação de rede que devem ser satisfeitos para operação apropriada. O control plane do cluster necessita alcançar os Pods do Metrics Server, tipicamente na porta 10250, para consultar a Metrics API agregada. Simultaneamente, o Metrics Server deve conseguir estabelecer conexões com o Kubelet em todos os nós do cluster, também na porta 10250.

Configurações de firewall, security groups, ou network policies que bloqueiem estas comunicações impedirão o funcionamento correto do Metrics Server. Em ambientes cloud, é essencial verificar que os security groups associados aos nós permitem tráfego na porta 10250 tanto de entrada quanto de saída.

A autenticação e autorização constituem aspectos críticos da segurança do Metrics Server. O componente requer permissões apropriadas através de RBAC (Role-Based Access Control) para consultar métricas dos Kubelets e expor a Metrics API. Os manifestos oficiais incluem configurações de ServiceAccount, ClusterRole, e ClusterRoleBinding que estabelecem as permissões mínimas necessárias.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  - nodes/stats
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
```

### 3.5. Validação e Troubleshooting

Após a instalação, é fundamental validar que o Metrics Server está operando corretamente e expondo métricas. Diversos comandos auxiliam nesta validação, permitindo identificar rapidamente problemas de configuração.

```bash
# Verificar status do Deployment do Metrics Server
kubectl get deployment metrics-server -n kube-system

# Examinar logs para identificar erros
kubectl logs -n kube-system deployment/metrics-server

# Verificar disponibilidade da Metrics API
kubectl get apiservices v1beta1.metrics.k8s.io -o yaml

# Testar consulta de métricas
kubectl top nodes
kubectl top pods -n kube-system

# Verificar conectividade com Kubelets
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
```

Problemas comuns incluem falhas de validação de certificados, impossibilidade de alcançar Kubelets devido a configurações de rede, ou permissões RBAC insuficientes. A análise dos logs do Metrics Server frequentemente revela a causa raiz de falhas operacionais, permitindo correções direcionadas.

## 4. Horizontal Pod Autoscaler: Versão 1

### 4.1. Introdução ao HPA v1

A versão 1 da API do Horizontal Pod Autoscaler representa a implementação inicial e mais simples do mecanismo de autoescalamento horizontal no Kubernetes. Embora limitada em comparação com versões posteriores, a v1 cobre adequadamente os casos de uso mais comuns, focando exclusivamente em escalamento baseado na utilização de CPU.

A simplicidade da v1 facilita a adoção inicial e compreensão dos conceitos fundamentais de autoescalamento. Administradores podem rapidamente implementar políticas básicas sem necessidade de dominar configurações complexas ou métricas customizadas. Esta característica torna a v1 apropriada para ambientes onde os requisitos de escalamento são diretos e a CPU constitui o indicador primário de carga.

### 4.2. Estrutura do Manifesto HPA v1

O manifesto de um HorizontalPodAutoscaler versão 1 segue uma estrutura declarativa simples, especificando o recurso alvo para escalamento, limites de réplicas, e o threshold de utilização de CPU que aciona ações de escalamento.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application-deployment
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

O campo `scaleTargetRef` identifica o recurso que será escalado, tipicamente um Deployment, mas podendo ser também um ReplicaSet ou StatefulSet. Os campos `minReplicas` e `maxReplicas` estabelecem os limites inferior e superior para o número de réplicas, prevenindo tanto subprovisioning quanto overprovisioning excessivo.

O `targetCPUUtilizationPercentage` define o percentual alvo de utilização de CPU. Este valor é interpretado em relação aos resource requests definidos nos containers dos Pods. Por exemplo, se um container especifica um request de 1000m (1 core) e o targetCPUUtilizationPercentage é 70%, o HPA tentará manter a utilização em aproximadamente 700m por réplica.

### 4.3. Algoritmo de Cálculo de Réplicas

O HPA implementa um algoritmo relativamente simples para determinar o número desejado de réplicas baseado na métrica de CPU. A fórmula fundamental é:

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

Por exemplo, considerando:
- Réplicas atuais: 5
- Utilização atual de CPU: 85%
- Utilização alvo de CPU: 70%

O cálculo resulta em:
```
desiredReplicas = ceil[5 * (85 / 70)] = ceil[6.07] = 7
```

O sistema escalaria de 5 para 7 réplicas. O uso da função ceiling garante arredondamento para cima, favorecendo disponibilidade sobre otimização de custos em situações limítrofes.

### 4.4. Importância dos Resource Requests

A funcionalidade do HPA v1 depende criticamente da definição apropriada de resource requests nos containers. O percentual de utilização de CPU é calculado em relação a estes requests, não em relação à capacidade total do nó ou aos limits definidos.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application-deployment
spec:
  replicas: 2
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
        image: application:latest
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        ports:
        - containerPort: 8080
```

A ausência de resource requests impede o funcionamento correto do HPA, pois o cálculo de percentual torna-se impossível. É uma prática recomendada sempre especificar requests que reflitam realisticamente os requisitos da aplicação em condições normais de operação.

### 4.5. Criação via kubectl autoscale

Além da aplicação de manifestos YAML, o HPA v1 pode ser criado através do comando `kubectl autoscale`, oferecendo uma interface mais imediata para casos simples.

```bash
# Criar HPA para um Deployment
kubectl autoscale deployment application-deployment \
  --min=2 \
  --max=10 \
  --cpu-percent=70 \
  --namespace=production

# Verificar status do HPA
kubectl get hpa application-deployment -n production

# Descrever detalhes do HPA
kubectl describe hpa application-deployment -n production
```

Esta abordagem imperativa é conveniente para testes rápidos ou ambientes de desenvolvimento, mas a prática recomendada para ambientes de produção consiste em manter manifestos YAML versionados em sistemas de controle de versão, permitindo rastreabilidade e reprodutibilidade das configurações.

### 4.6. Limitações da Versão 1

A API v1 do HPA apresenta limitações significativas que motivaram o desenvolvimento de versões posteriores. A restrição a CPU como única métrica de escalamento constitui a limitação mais evidente. Muitas aplicações experimentam gargalos em outros recursos, como memória, conexões de rede, ou métricas específicas da aplicação que não se correlacionam diretamente com utilização de CPU.

A ausência de controle granular sobre o comportamento de escalamento representa outra limitação. Não há mecanismos para ajustar a velocidade de scale-up ou scale-down, definir janelas de estabilização customizadas, ou implementar políticas diferenciadas para escalamento ascendente versus descendente. Estas capacidades tornam-se disponíveis apenas na versão 2 da API.

## 5. Horizontal Pod Autoscaler: Versão 2

### 5.1. Evolução e Capacidades Expandidas

A versão 2 da API do Horizontal Pod Autoscaler (autoscaling/v2) introduz capacidades substancialmente ampliadas em relação à v1, mantendo retrocompatibilidade com configurações existentes. As melhorias abrangem suporte a múltiplas métricas simultâneas, métricas customizadas e externas, controle refinado sobre comportamento de escalamento, e expressividade significativamente maior nas políticas de autoescalamento.

Esta evolução reflete as necessidades de ambientes de produção complexos, onde decisões de escalamento devem considerar múltiplos fatores simultaneamente e adaptar-se a características específicas de diferentes tipos de aplicações. A v2 torna-se a versão recomendada para todos os novos deployments, oferecendo flexibilidade necessária para implementar estratégias sofisticadas de autoescalamento.

### 5.2. Estrutura Básica do Manifesto HPA v2

O manifesto HPA v2 mantém estrutura similar à v1, mas substitui o campo `targetCPUUtilizationPercentage` por uma seção `metrics` mais expressiva que aceita array de definições de métricas.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa-v2
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application-deployment
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 5.3. Tipos de Métricas

A v2 suporta quatro tipos distintos de métricas, cada um apropriado para diferentes cenários de escalamento.

#### 5.3.1. Resource Metrics

Resource metrics referem-se aos recursos básicos de CPU e memória, coletados automaticamente pelo Metrics Server. São as métricas mais comumente utilizadas e não requerem infraestrutura adicional.

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 75
- type: Resource
  resource:
    name: memory
    target:
      type: AverageValue
      averageValue: 512Mi
```

O campo `type` dentro de `target` determina como a métrica é interpretada:
- `Utilization`: Percentual em relação ao resource request
- `AverageValue`: Valor absoluto médio entre todas as réplicas
- `Value`: Valor total agregado (menos comum para resource metrics)

#### 5.3.2. Container Resource Metrics

Permite escalamento baseado em recursos de containers específicos dentro de Pods multi-container, oferecendo granularidade adicional.

```yaml
metrics:
- type: ContainerResource
  containerResource:
    name: cpu
    container: sidecar-container
    target:
      type: Utilization
      averageUtilization: 60
```

#### 5.3.3. Pods Metrics

Custom metrics associadas diretamente aos Pods, tipicamente coletadas por adaptadores de métricas customizadas como o Prometheus Adapter.

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

#### 5.3.4. Object Metrics

Métricas associadas a outros objetos do Kubernetes, como Ingress ou Services, permitindo escalamento baseado em características externas aos Pods.

```yaml
metrics:
- type: Object
  object:
    metric:
      name: requests_per_second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: application-ingress
    target:
      type: Value
      value: "10000"
```

#### 5.3.5. External Metrics

Métricas provenientes de sistemas externos ao cluster, como serviços de monitoramento cloud, filas de mensagens, ou APIs externas.

```yaml
metrics:
- type: External
  external:
    metric:
      name: queue_messages_ready
      selector:
        matchLabels:
          queue_name: orders-queue
    target:
      type: AverageValue
      averageValue: "30"
```

### 5.4. Múltiplas Métricas e Lógica de Decisão

Quando múltiplas métricas são especificadas, o HPA calcula o número desejado de réplicas para cada métrica individualmente e seleciona o maior valor resultante. Esta abordagem conservadora garante que o sistema escala para atender todas as condições simultaneamente.

Por exemplo, considerando:
- Métrica de CPU sugere 5 réplicas
- Métrica de memória sugere 7 réplicas
- Métrica customizada sugere 6 réplicas

O HPA escalará para 7 réplicas, garantindo que todas as métricas fiquem dentro de seus targets desejados. Esta lógica previne situações onde otimização para uma métrica resultaria em violação de constraints de outra.

### 5.5. Comportamento de Escalamento: Policies

A v2 introduz a seção `behavior` que permite controle granular sobre como e quando o escalamento ocorre, diferenciando políticas para scale-up e scale-down.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa-advanced
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application-deployment
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

#### 5.5.1. Janela de Estabilização

O `stabilizationWindowSeconds` define um período durante o qual o HPA observa as métricas antes de tomar decisões de escalamento. Para scale-down, um valor maior (como 300 segundos) previne oscilações frequentes, garantindo que reduções de carga sejam sustentadas antes de remover réplicas. Para scale-up, valores menores ou zero permitem resposta rápida a aumentos de demanda.

#### 5.5.2. Políticas de Escalamento

As políticas definem a taxa máxima de mudança permitida. Cada política especifica:
- `type`: Pode ser `Pods` (número absoluto) ou `Percent` (percentual das réplicas atuais)
- `value`: O valor da mudança permitida
- `periodSeconds`: O período de tempo sobre o qual a política se aplica

O campo `selectPolicy` determina como múltiplas políticas são combinadas:
- `Max`: Seleciona a política que permite mais mudança (apropriado para scale-up)
- `Min`: Seleciona a política que permite menos mudança (apropriado para scale-down)
- `Disabled`: Desabilita o escalamento naquela direção

### 5.6. Exemplo Completo de Configuração Avançada

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-application-hpa
  namespace: production
  labels:
    app: web-application
    environment: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-application
  minReplicas: 5
  maxReplicas: 50
  metrics:
  # CPU como métrica primária
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  # Memória como métrica secundária
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Métrica customizada de requisições por segundo
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      # Permite dobrar o número de réplicas a cada 30 segundos
      - type: Percent
        value: 100
        periodSeconds: 30
      # Ou adicionar até 8 Pods a cada 30 segundos
      - type: Pods
        value: 8
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
      # Reduz no máximo 10% das réplicas a cada 60 segundos
      - type: Percent
        value: 10
        periodSeconds: 60
      # Ou remove no máximo 2 Pods a cada 60 segundos
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

## 6. Testes de Carga e Validação

### 6.1. Importância dos Testes de Estresse

A validação efetiva de configurações de HPA requer simulação realista de condições operacionais através de testes de carga controlados. Estes testes permitem observar o comportamento do sistema sob diferentes níveis de demanda, identificar gargalos, validar tempos de resposta do autoescalamento, e ajustar configurações antes da exposição a tráfego de produção real.

Testes inadequados ou ausentes frequentemente resultam em configurações de HPA subótimas que falham em responder apropriadamente durante incidentes reais de produção. A calibração correta de thresholds de escalamento, limites de réplicas, e políticas de comportamento depende fundamentalmente de dados empíricos obtidos através de testes sistemáticos.

### 6.2. Ferramentas de Teste de Carga

#### 6.2.1. FortIO

FortIO constitui uma ferramenta especializada de teste de carga e performance, originalmente desenvolvida pelo projeto Istio. Oferece capacidade de gerar cargas HTTP, HTTPS e gRPC com controle preciso sobre taxa de requisições, concorrência, duração, e distribuição temporal.

```bash
# Instalação do FortIO
kubectl create namespace load-testing
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio
  namespace: load-testing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio
  template:
    metadata:
      labels:
        app: fortio
    spec:
      containers:
      - name: fortio
        image: fortio/fortio:latest
        ports:
        - containerPort: 8080
        - containerPort: 8079
EOF

# Execução de teste básico
kubectl exec -n load-testing deployment/fortio -- \
  fortio load -qps 1000 -t 60s -c 50 \
  http://application-service.production.svc.cluster.local:8080/

# Teste com relatório detalhado
kubectl exec -n load-testing deployment/fortio -- \
  fortio load -qps 2000 -t 300s -c 100 -json /tmp/results.json \
  http://application-service.production.svc.cluster.local:8080/api/endpoint
```

Parâmetros importantes:
- `-qps`: Queries Per Second desejadas
- `-t`: Duração do teste
- `-c`: Número de conexões concorrentes
- `-json`: Exportar resultados em formato JSON para análise posterior

#### 6.2.2. K6

K6 é uma ferramenta moderna de teste de carga com suporte nativo a scripting em JavaScript, permitindo cenários complexos e realistas.

```javascript
// script-k6.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp-up
    { duration: '5m', target: 100 },  // Stay at peak
    { duration: '2m', target: 200 },  // Spike
    { duration: '5m', target: 200 },  // Stay at spike
    { duration: '3m', target: 0 },    // Ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],   // Error rate under 1%
  },
};

export default function () {
  const res = http.get('http://application-service.production.svc.cluster.local:8080/api/data');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

```bash
# Execução do teste K6
k6 run --vus 100 --duration 10m script-k6.js
```

#### 6.2.3. Apache JMeter

JMeter oferece interface gráfica completa e capacidades extensivas de configuração, apropriado para cenários complexos de teste.

```bash
# Execução headless do JMeter
jmeter -n -t test-plan.jmx -l results.jtl -e -o ./report
```

### 6.3. Cenários de Teste

#### 6.3.1. Teste de Ramp-Up Gradual

Simula crescimento gradual de carga, permitindo observar como o HPA responde a aumentos progressivos de demanda.

```bash
# Teste progressivo com FortIO
for qps in 100 500 1000 2000 5000; do
  echo "Testing with ${qps} QPS"
  kubectl exec -n load-testing deployment/fortio -- \
    fortio load -qps ${qps} -t 180s -c 50 \
    http://application-service.production.svc.cluster.local:8080/
  echo "Waiting for stabilization..."
  sleep 120
done
```

#### 6.3.2. Teste de Spike Súbito

Simula picos súbitos de tráfego, validando a capacidade de resposta rápida do sistema.

```bash
# Spike test: baixa carga seguida de pico abrupto
kubectl exec -n load-testing deployment/fortio -- \
  fortio load -qps 100 -t 60s -c 10 \
  http://application-service.production.svc.cluster.local:8080/

# Espera breve
sleep 10

# Pico súbito
kubectl exec -n load-testing deployment/fortio -- \
  fortio load -qps 5000 -t 300s -c 200 \
  http://application-service.production.svc.cluster.local:8080/
```

#### 6.3.3. Teste de Sustentação

Mantém carga elevada por período prolongado, verificando estabilidade e identificando memory leaks ou degradação progressiva.

```bash
# Teste de longa duração
kubectl exec -n load-testing deployment/fortio -- \
  fortio load -qps 2000 -t 3600s -c 100 \
  http://application-service.production.svc.cluster.local:8080/
```

### 6.4. Monitoramento Durante Testes

Durante a execução de testes de carga, é essencial monitorar múltiplas métricas simultaneamente para correlacionar comportamento do HPA com performance da aplicação.

```bash
# Terminal 1: Monitorar status do HPA
watch -n 2 'kubectl get hpa -n production'

# Terminal 2: Monitorar Pods
watch -n 2 'kubectl get pods -n production -l app=application'

# Terminal 3: Monitorar métricas de recursos
watch -n 5 'kubectl top pods -n production -l app=application'

# Terminal 4: Logs da aplicação
kubectl logs -n production -l app=application -f --tail=100

# Terminal 5: Eventos do cluster
kubectl get events -n production --watch --sort-by='.lastTimestamp'
```

### 6.5. Análise de Resultados

A análise sistemática dos resultados requer correlação entre múltiplas fontes de dados.

```bash
# Extrair histórico de escalamento do HPA
kubectl describe hpa application-hpa -n production | grep -A 20 "Events:"

# Analisar latências percentis
kubectl exec -n load-testing deployment/fortio -- \
  fortio report -json /tmp/results.json

# Verificar distribuição de carga entre réplicas
for pod in $(kubectl get pods -n production -l app=application -o name); do
  echo "Pod: $pod"
  kubectl exec -n production $pod -- cat /proc/loadavg
done
```

Métricas chave para análise:
- Tempo de resposta (p50, p95, p99)
- Taxa de erro
- Throughput alcançado versus esperado
- Tempo entre detecção de carga elevada e provisionamento de réplicas
- Tempo de warm-up de novas réplicas
- Estabilidade após scale-down

### 6.6. Ajuste de Configurações

Baseado nos resultados dos testes, iterações nas configurações frequentemente são necessárias:

```yaml
# Exemplo de ajustes baseados em observações
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa-tuned
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application-deployment
  # Aumentado minReplicas baseado em carga base observada
  minReplicas: 4
  # Aumentado maxReplicas baseado em picos observados
  maxReplicas: 25
  metrics:
  # Threshold de CPU ajustado para responder mais cedo
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60  # Reduzido de 70%
  behavior:
    scaleUp:
      # Reduzido stabilization para resposta mais rápida
      stabilizationWindowSeconds: 15
      policies:
      # Aumentado agressividade de scale-up
      - type: Pods
        value: 6  # Aumentado de 4
        periodSeconds: 15
```

## 7. Cenários Avançados e Otimizações

### 7.1. Aplicações com Processamento Intensivo

Aplicações que realizam operações computacionalmente intensivas, como processamento de imagens, encoding de vídeo, ou operações criptográficas, frequentemente exibem padrões de consumo de recursos distintos que requerem configurações especializadas.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: video-processing-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: video-processor
  minReplicas: 3
  maxReplicas: 30
  metrics:
  # CPU como métrica principal para workloads computacionais
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
  # Métrica customizada baseada em fila de jobs
  - type: Pods
    pods:
      metric:
        name: jobs_in_queue
      target:
        type: AverageValue
        averageValue: "5"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      # Scale-up agressivo para processar backlog rapidamente
      - type: Percent
        value: 150
        periodSeconds: 20
    scaleDown:
      # Scale-down conservador para evitar re-escalamento frequente
      stabilizationWindowSeconds: 600
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
```

### 7.2. Aplicações com Operações de I/O Intensivas

Aplicações que realizam operações intensivas de I/O, como escrita em arquivos ou processamento de streams, podem não demonstrar alta utilização de CPU apesar de estarem sob carga significativa.

```typescript
// Exemplo de aplicação com I/O intensivo em Node.js
import { createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';
import express from 'express';

const app = express();

app.post('/process-data', async (req, res) => {
  const startTime = Date.now();
  const outputStream = createWriteStream(`/tmp/output-${startTime}.txt`);
  
  try {
    // Operação de I/O que pode não gerar alta CPU
    await pipeline(
      req,
      // Transform streams aqui
      outputStream
    );
    
    res.status(200).json({ 
      status: 'success',
      processingTime: Date.now() - startTime 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(8080);
```

Para estas aplicações, métricas customizadas baseadas em taxa de requisições ou latência podem ser mais apropriadas:

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "100"
- type: Pods
  pods:
    metric:
      name: avg_request_duration_ms
    target:
      type: AverageValue
      averageValue: "500"
```

### 7.3. Warm-up e Readiness

Aplicações com tempo de inicialização significativo requerem configurações que considerem o período de warm-up antes que novos Pods possam processar requisições efetivamente.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-application
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: java-app
        image: java-app:latest
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 90
          periodSeconds: 20
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
```

Configuração HPA correspondente com scale-up mais agressivo para compensar warm-up:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    # Scale-up mais agressivo considerando tempo de warm-up
    - type: Pods
      value: 5
      periodSeconds: 30
```

### 7.4. Integração com Service Mesh

Em ambientes que utilizam service mesh como Istio ou Linkerd, métricas adicionais de rede tornam-se disponíveis para decisões de escalamento.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: service-mesh-aware-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: microservice
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # Métricas tradicionais
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Métricas do service mesh via Prometheus Adapter
  - type: Pods
    pods:
      metric:
        name: istio_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"
  - type: Pods
    pods:
      metric:
        name: istio_request_duration_p99
      target:
        type: AverageValue
        averageValue: "1000"  # 1000ms
```

### 7.5. Estratégias Multi-Cluster

Em ambientes multi-cluster, o escalamento pode ser coordenado através de ferramentas como Kubernetes Federation ou soluções customizadas.

```yaml
# Configuração por cluster com limites apropriados
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa-cluster-us-east
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application
  minReplicas: 5
  maxReplicas: 30
  # Configurações específicas para cluster primário
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa-cluster-eu-west
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application
  minReplicas: 3
  maxReplicas: 20
  # Configurações para cluster secundário
```

## 8. Troubleshooting e Diagnóstico

### 8.1. HPA Não Está Escalando

Quando o HPA falha em escalar conforme esperado, uma abordagem sistemática de diagnóstico é necessária.

```bash
# Verificar status detalhado do HPA
kubectl describe hpa <hpa-name> -n <namespace>

# Verificar se métricas estão disponíveis
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods

# Verificar logs do Metrics Server
kubectl logs -n kube-system deployment/metrics-server

# Verificar eventos do namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Verificar se os resource requests estão definidos
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

Causas comuns:
- Metrics Server não instalado ou não operacional
- Resource requests não definidos nos containers
- Valores de threshold muito altos ou baixos
- Métricas customizadas não sendo expostas corretamente
- Limites de maxReplicas já atingidos

### 8.2. Oscilação de Réplicas (Flapping)

Oscilação ocorre quando o HPA escala up e down repetidamente em intervalos curtos.

```yaml
# Solução: Ajustar janelas de estabilização
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300
    
# Solução: Ajustar thresholds para criar hysteresis
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      # Use valor mais conservador
      averageUtilization: 60  # Ao invés de 80%
```

### 8.3. Escalamento Muito Lento

Quando o sistema não responde rapidamente o suficiente a picos de carga:

```yaml
behavior:
  scaleUp:
    # Reduzir ou eliminar janela de estabilização
    stabilizationWindowSeconds: 0
    policies:
    # Aumentar agressividade
    - type: Percent
      value: 200  # Dobra ou triplica rapidamente
      periodSeconds: 15
    - type: Pods
      value: 10  # Adiciona muitos Pods rapidamente
      periodSeconds: 15
    selectPolicy: Max

# Reduzir threshold para acionar mais cedo
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50  # Aciona mais cedo
```

### 8.4. Métricas Customizadas Não Funcionam

```bash
# Verificar se Custom Metrics API está registrada
kubectl get apiservice v1beta1.custom.metrics.k8s.io -o yaml

# Verificar se o adapter está rodando (ex: Prometheus Adapter)
kubectl get pods -n monitoring -l app=prometheus-adapter

# Testar consulta de métrica customizada
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/<namespace>/pods/*/http_requests_per_second"

# Verificar configuração do adapter
kubectl get configmap prometheus-adapter -n monitoring -o yaml
```

## 9. Melhores Práticas

### 9.1. Definição de Resource Requests e Limits

```yaml
# Prática recomendada para production
resources:
  requests:
    # Request baseado em uso médio observado
    cpu: 500m
    memory: 512Mi
  limits:
    # Limit com margem para picos, mas não excessivo
    cpu: 1000m
    memory: 1Gi
```

### 9.2. Configuração de Réplicas Mínimas

```yaml
# Garantir disponibilidade mesmo sem carga
minReplicas: 3  # Nunca menos que 2 para produção

# Para ambientes críticos
minReplicas: 5  # Garante capacidade mesmo com falhas de nós
```

### 9.3. Múltiplas Métricas para Robustez

```yaml
metrics:
# Usar múltiplas métricas para decisões mais informadas
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80
- type: Pods
  pods:
    metric:
      name: application_specific_metric
    target:
      type: AverageValue
      averageValue: "1000"
```

### 9.4. Monitoramento e Alertas

```yaml
# Exemplo de PrometheusRule para alertas
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: hpa-alerts
spec:
  groups:
  - name: hpa
    rules:
    - alert: HPAMaxedOut
      expr: |
        kube_horizontalpodautoscaler_status_current_replicas
        >= kube_horizontalpodautoscaler_spec_max_replicas
      for: 15m
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} está no máximo de réplicas"
    - alert: HPAScalingDisabled
      expr: |
        kube_horizontalpodautoscaler_status_condition{condition="ScalingActive",status="false"}
      for: 5m
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} não está escalando"
```

### 9.5. Documentação de Configurações

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa
  annotations:
    # Documentar decisões de configuração
    description: "HPA para aplicação web principal"
    tuning-notes: "Threshold CPU ajustado baseado em teste de carga de 2024-01"
    owner: "team-platform"
    review-date: "2024-06-01"
spec:
  # Configurações...
```

## 10. Conclusões

O Horizontal Pod Autoscaler representa um componente essencial para operação eficiente e resiliente de aplicações no Kubernetes. A capacidade de adaptar automaticamente a capacidade computacional às demandas flutuantes não apenas melhora a experiência dos usuários finais através de tempos de resposta consistentes, mas também otimiza significativamente os custos operacionais ao evitar o provisionamento excessivo de recursos.

A evolução da API de autoscaling/v1 para autoscaling/v2 demonstra a maturidade crescente do ecossistema Kubernetes, oferecendo flexibilidade e controle que atendem desde cenários simples até arquiteturas complexas de microsserviços. A integração nativa com o Metrics Server, combinada com extensibilidade através de métricas customizadas e externas, proporciona uma plataforma robusta para implementação de estratégias sofisticadas de escalamento que consideram múltiplos fatores simultaneamente.

A eficácia do HPA, entretanto, depende fundamentalmente de configuração apropriada baseada em compreensão profunda do comportamento da aplicação. Testes sistemáticos de carga, análise criteriosa de métricas, e iteração contínua das configurações constituem práticas essenciais para alcançar resultados ótimos. A definição adequada de resource requests, thresholds de escalamento, políticas de comportamento, e limites de réplicas requer balanceamento cuidadoso entre responsividade, estabilidade, e eficiência de custos.

Ambientes de produção modernos beneficiam-se imensamente da combinação de escalabilidade horizontal automática com outras práticas de engenharia de confiabilidade, incluindo circuit breakers, rate limiting, chaos engineering, e observabilidade abrangente. O HPA não opera isoladamente, mas como parte de um ecossistema mais amplo de ferramentas e práticas que coletivamente garantem disponibilidade, performance, e resiliência das aplicações.

À medida que o ecossistema Kubernetes continua evoluindo, novas capacidades de escalamento emergem, incluindo escalamento preditivo baseado em machine learning, integração mais profunda com plataformas de observabilidade, e coordenação multi-cluster mais sofisticada. A compreensão sólida dos fundamentos apresentados neste documento fornece a base necessária para explorar estas tecnologias emergentes e adaptar estratégias de escalamento às necessidades específicas de cada contexto operacional.

## 11. Referências Bibliográficas

### 11.1. Documentação Oficial do Kubernetes

- Kubernetes Documentation. "Horizontal Pod Autoscaler". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

- Kubernetes Documentation. "Horizontal Pod Autoscaler Walkthrough". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

- Kubernetes Documentation. "Metrics Server". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

- Kubernetes API Reference. "HorizontalPodAutoscaler v2 autoscaling". Kubernetes.io. Disponível em: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/

### 11.2. Repositórios Oficiais

- Kubernetes SIGs. "Metrics Server". GitHub. Disponível em: https://github.com/kubernetes-sigs/metrics-server

- Kubernetes. "Community Documentation - HPA". GitHub. Disponível em: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/autoscaling/horizontal-pod-autoscaler.md

### 11.3. Ferramentas de Teste e Monitoramento

- Fortio Project. "Fortio Load Testing Library". Fortio.org. Disponível em: https://fortio.org/

- Grafana k6. "k6 Documentation". k6.io. Disponível em: https://k6.io/docs/

- Apache Software Foundation. "Apache JMeter". JMeter.apache.org. Disponível em: https://jmeter.apache.org/

### 11.4. KEDA - Kubernetes Event-Driven Autoscaling

- KEDA. "KEDA Documentation". KEDA.sh. Disponível em: https://keda.sh/docs/

### 11.5. Prometheus e Métricas Customizadas

- Prometheus. "Prometheus Documentation". Prometheus.io. Disponível em: https://prometheus.io/docs/

- Kubernetes SIGs. "Prometheus Adapter for Kubernetes Metrics APIs". GitHub. Disponível em: https://github.com/kubernetes-sigs/prometheus-adapter

### 11.6. Artigos e Recursos Adicionais

- Cloud Native Computing Foundation. "Autoscaling in Kubernetes". CNCF.io. Disponível em: https://www.cncf.io/

- Kubernetes Blog. "HorizontalPodAutoscaler Improvements". Kubernetes.io/blog. Disponível em: https://kubernetes.io/blog/

## 12. Apêndices

### Apêndice A: Comandos Úteis de Diagnóstico

```bash
# Verificação completa do HPA
kubectl get hpa --all-namespaces
kubectl describe hpa <nome-hpa> -n <namespace>

# Verificação de métricas
kubectl top nodes
kubectl top pods --all-namespaces
kubectl top pods -n <namespace> --sort-by=cpu
kubectl top pods -n <namespace> --sort-by=memory

# Verificação do Metrics Server
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=100

# Verificação de APIs de métricas
kubectl get apiservices | grep metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1

# Consulta direta de métricas de nós
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq .

# Consulta direta de métricas de Pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods | jq .

# Verificação de eventos relacionados a escalamento
kubectl get events -n <namespace> --field-selector involvedObject.kind=HorizontalPodAutoscaler

# Monitoramento contínuo de HPA
watch -n 2 'kubectl get hpa -n <namespace>'

# Exportação de configuração do HPA
kubectl get hpa <nome-hpa> -n <namespace> -o yaml > hpa-backup.yaml

# Simulação de dry-run de alterações
kubectl apply -f hpa-config.yaml --dry-run=client

# Verificação de histórico de escalamento
kubectl describe hpa <nome-hpa> -n <namespace> | grep -A 30 "Events:"
```

### Apêndice B: Exemplo de Deployment Completo com HPA

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production-app
  labels:
    environment: production

---
# ConfigMap com configurações da aplicação
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production-app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "1000"

---
# Deployment da aplicação
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
  namespace: production-app
  labels:
    app: web-application
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-application
  template:
    metadata:
      labels:
        app: web-application
        version: v1.0.0
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-application
              topologyKey: kubernetes.io/hostname
      containers:
      - name: application
        image: myregistry/web-application:1.0.0
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-application-service
  namespace: production-app
  labels:
    app: web-application
spec:
  type: ClusterIP
  selector:
    app: web-application
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http

---
# HorizontalPodAutoscaler v2
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-application-hpa
  namespace: production-app
  annotations:
    description: "HPA para aplicação web com múltiplas métricas"
    owner: "platform-team"
    last-tuned: "2024-01-15"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-application
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 5
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min

---
# PodDisruptionBudget para garantir disponibilidade durante manutenções
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-application-pdb
  namespace: production-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-application
```

### Apêndice C: Script de Teste de Carga Automatizado

```bash
#!/bin/bash
# script-teste-carga.sh
# Script para execução automatizada de testes de carga e coleta de métricas

set -e

NAMESPACE="production-app"
HPA_NAME="web-application-hpa"
SERVICE_URL="http://web-application-service.${NAMESPACE}.svc.cluster.local"
RESULTS_DIR="./test-results-$(date +%Y%m%d-%H%M%S)"

# Criar diretório de resultados
mkdir -p "${RESULTS_DIR}"

echo "Iniciando testes de carga..."
echo "Namespace: ${NAMESPACE}"
echo "HPA: ${HPA_NAME}"
echo "Resultados em: ${RESULTS_DIR}"

# Função para capturar estado do HPA
capture_hpa_state() {
    local timestamp=$1
    kubectl get hpa ${HPA_NAME} -n ${NAMESPACE} -o yaml > "${RESULTS_DIR}/hpa-state-${timestamp}.yaml"
    kubectl describe hpa ${HPA_NAME} -n ${NAMESPACE} > "${RESULTS_DIR}/hpa-describe-${timestamp}.txt"
}

# Função para capturar métricas de Pods
capture_pod_metrics() {
    local timestamp=$1
    kubectl top pods -n ${NAMESPACE} -l app=web-application > "${RESULTS_DIR}/pod-metrics-${timestamp}.txt"
    kubectl get pods -n ${NAMESPACE} -l app=web-application -o wide > "${RESULTS_DIR}/pod-list-${timestamp}.txt"
}

# Estado inicial
echo "Capturando estado inicial..."
capture_hpa_state "initial"
capture_pod_metrics "initial"

# Teste 1: Carga baixa (baseline)
echo "Executando teste 1: Carga baixa (100 QPS)..."
kubectl exec -n load-testing deployment/fortio -- \
    fortio load -qps 100 -t 180s -c 10 -json /tmp/test1.json \
    ${SERVICE_URL} || true

sleep 60
capture_hpa_state "test1-post"
capture_pod_metrics "test1-post"

# Teste 2: Carga média (ramp-up gradual)
echo "Executando teste 2: Carga média (1000 QPS)..."
kubectl exec -n load-testing deployment/fortio -- \
    fortio load -qps 1000 -t 300s -c 50 -json /tmp/test2.json \
    ${SERVICE_URL} || true

sleep 60
capture_hpa_state "test2-post"
capture_pod_metrics "test2-post"

# Teste 3: Pico de carga
echo "Executando teste 3: Pico de carga (5000 QPS)..."
kubectl exec -n load-testing deployment/fortio -- \
    fortio load -qps 5000 -t 300s -c 200 -json /tmp/test3.json \
    ${SERVICE_URL} || true

sleep 60
capture_hpa_state "test3-post"
capture_pod_metrics "test3-post"

# Teste 4: Sustentação de carga alta
echo "Executando teste 4: Sustentação (3000 QPS por 600s)..."
kubectl exec -n load-testing deployment/fortio -- \
    fortio load -qps 3000 -t 600s -c 150 -json /tmp/test4.json \
    ${SERVICE_URL} || true

sleep 60
capture_hpa_state "test4-post"
capture_pod_metrics "test4-post"

# Período de cool-down
echo "Aguardando cool-down (10 minutos)..."
sleep 600

capture_hpa_state "final"
capture_pod_metrics "final"

# Capturar eventos
echo "Capturando eventos do namespace..."
kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp' > "${RESULTS_DIR}/events.txt"

# Gerar relatório resumido
echo "Gerando relatório resumido..."
cat > "${RESULTS_DIR}/REPORT.md" << EOF
# Relatório de Testes de Carga

**Data:** $(date)
**Namespace:** ${NAMESPACE}
**HPA:** ${HPA_NAME}

## Testes Executados

### Teste 1: Carga Baixa
- QPS: 100
- Duração: 180s
- Concorrência: 10

### Teste 2: Carga Média
- QPS: 1000
- Duração: 300s
- Concorrência: 50

### Teste 3: Pico de Carga
- QPS: 5000
- Duração: 300s
- Concorrência: 200

### Teste 4: Sustentação
- QPS: 3000
- Duração: 600s
- Concorrência: 150

## Arquivos Gerados

- Estado do HPA em diferentes momentos
- Métricas de Pods em diferentes momentos
- Eventos do namespace
- Resultados detalhados dos testes FortIO

## Próximos Passos

1. Analisar os arquivos de métricas
2. Correlacionar tempo de resposta com escalamento
3. Ajustar configurações do HPA se necessário
4. Documentar observações e decisões
EOF

echo "Testes concluídos!"
echo "Resultados disponíveis em: ${RESULTS_DIR}"
echo "Revise o arquivo ${RESULTS_DIR}/REPORT.md para análise inicial"
```

### Apêndice D: Exemplo de Aplicação Node.js com Métricas

```typescript
// server.ts
import express, { Request, Response } from 'express';
import { createWriteStream } from 'fs';
import { register, Counter, Histogram, Gauge } from 'prom-client';

const app = express();
const PORT = process.env.PORT || 8080;

// Métricas Prometheus
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total de requisições HTTP',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_ms',
  help: 'Duração das requisições HTTP em milissegundos',
  labelNames: ['method', 'route', 'status'],
  buckets: [10, 50, 100, 200, 500, 1000, 2000, 5000]
});

const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Número de conexões ativas'
});

// Middleware para rastrear métricas
app.use((req: Request, res: Response, next) => {
  activeConnections.inc();
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const route = req.route?.path || req.path;
    
    httpRequestsTotal.inc({
      method: req.method,
      route: route,
      status: res.statusCode
    });
    
    httpRequestDuration.observe({
      method: req.method,
      route: route,
      status: res.statusCode
    }, duration);
    
    activeConnections.dec();
  });

  next();
});

// Endpoints de saúde
app.get('/health/live', (req: Request, res: Response) => {
  res.status(200).json({ status: 'alive' });
});

app.get('/health/ready', (req: Request, res: Response) => {
  res.status(200).json({ status: 'ready' });
});

app.get('/health/startup', (req: Request, res: Response) => {
  res.status(200).json({ status: 'started' });
});

// Endpoint de métricas para Prometheus
app.get('/metrics', async (req: Request, res: Response) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Endpoint simples
app.get('/api/simple', (req: Request, res: Response) => {
  res.json({ message: 'Hello from simple endpoint', timestamp: Date.now() });
});

// Endpoint com processamento de CPU
app.get('/api/compute', (req: Request, res: Response) => {
  const iterations = 1000000;
  let result = 0;
  
  for (let i = 0; i < iterations; i++) {
    result += Math.sqrt(i) * Math.random();
  }
  
  res.json({ result, iterations, timestamp: Date.now() });
});

// Endpoint com I/O intensivo
app.post('/api/write', (req: Request, res: Response) => {
  const filename = `/tmp/output-${Date.now()}.txt`;
  const stream = createWriteStream(filename);
  
  const dataSize = 1000000; // 1MB de dados
  const chunk = 'A'.repeat(1000);
  
  for (let i = 0; i < dataSize / 1000; i++) {
    stream.write(chunk);
  }
  
  stream.end(() => {
    res.json({ 
      message: 'Data written successfully',
      filename,
      size: dataSize
    });
  });
});

// Endpoint com latência variável
app.get('/api/variable-latency', async (req: Request, res: Response) => {
  const delay = Math.floor(Math.random() * 500) + 100; // 100-600ms
  await new Promise(resolve => setTimeout(resolve, delay));
  res.json({ delay, timestamp: Date.now() });
});

// Iniciar servidor
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Health checks available at:`);
  console.log(`  - http://localhost:${PORT}/health/live`);
  console.log(`  - http://localhost:${PORT}/health/ready`);
  console.log(`  - http://localhost:${PORT}/health/startup`);
  console.log(`Metrics available at http://localhost:${PORT}/metrics`);
});
```

### Apêndice E: Glossário e Termos Técnicos

**Autoscaling (Autoescalamento)**: Processo automatizado de ajustar recursos computacionais alocados a uma aplicação baseado em métricas de utilização ou outros critérios definidos.

**Average Utilization**: Percentual médio de utilização de um recurso (CPU ou memória) calculado em relação aos resource requests definidos nos containers.

**Average Value**: Valor médio absoluto de uma métrica calculado entre todas as réplicas de uma aplicação.

**Behavior**: Seção do manifesto HPA v2 que define políticas específicas de escalamento ascendente e descendente, incluindo janelas de estabilização e taxas de mudança.

**Container Resource Metrics**: Métricas de recursos específicas de containers individuais dentro de Pods multi-container.

**Control Plane**: Conjunto de componentes do Kubernetes responsáveis pela gestão do cluster, incluindo API Server, Controller Manager, Scheduler e etcd.

**Custom Metrics**: Métricas específicas da aplicação, além das métricas padrão de recursos, tipicamente expostas através de adaptadores como Prometheus Adapter.

**Deployment**: Recurso do Kubernetes que gerencia ReplicaSets e fornece atualizações declarativas para Pods e ReplicaSets.

**Desired Replicas**: Número de réplicas que o HPA calcula como necessário para manter as métricas dentro dos targets definidos.

**Downscale (Scale-down)**: Processo de reduzir o número de réplicas ou recursos alocados a uma aplicação.

**External Metrics**: Métricas provenientes de sistemas externos ao cluster Kubernetes, como serviços cloud ou aplicações de terceiros.

**Flapping**: Oscilação rápida e repetida entre estados de escalamento, geralmente causada por configurações inadequadas de thresholds ou ausência de janelas de estabilização.

**Horizontal Scaling**: Escalabilidade implementada através do aumento ou diminuição do número de instâncias de uma aplicação.

**HPA (Horizontal Pod Autoscaler)**: Componente do Kubernetes que ajusta automaticamente o número de réplicas de Pods baseado em métricas observadas.

**KEDA (Kubernetes Event-Driven Autoscaling)**: Projeto que estende as capacidades de autoescalamento do Kubernetes para suportar escalamento baseado em eventos de diversas fontes.

**Kubelet**: Agente que executa em cada nó do cluster Kubernetes, responsável por garantir que containers estejam executando em Pods.

**Limits**: Quantidade máxima de recursos (CPU, memória) que um container pode utilizar, configurado no manifesto do Pod.

**Metrics API**: API agregada do Kubernetes que expõe métricas de recursos coletadas pelo Metrics Server.

**Metrics Server**: Componente do Kubernetes responsável por coletar e agregar métricas de recursos de nós e Pods.

**Object Metrics**: Métricas associadas a objetos específicos do Kubernetes, como Ingress ou Services.

**Pod**: Menor unidade deployável no Kubernetes, consistindo em um ou mais containers que compartilham recursos e rede.

**Pods Metrics**: Métricas customizadas associadas diretamente aos Pods, geralmente expostas pela aplicação.

**QPS (Queries Per Second)**: Métrica que representa o número de requisições processadas por segundo.

**RBAC (Role-Based Access Control)**: Sistema de controle de acesso do Kubernetes baseado em roles e permissões.

**Readiness Probe**: Verificação periódica para determinar se um container está pronto para receber tráfego.

**Replica**: Instância individual de um Pod gerenciado por um controlador como Deployment ou ReplicaSet.

**Requests**: Quantidade de recursos (CPU, memória) que um container declara necessitar, utilizada pelo scheduler para decisões de placement.

**Resource Metrics**: Métricas padrão de CPU e memória coletadas automaticamente pelo Metrics Server.

**Scale Target Reference**: Referência ao recurso (Deployment, ReplicaSet, StatefulSet) que o HPA controlará.

**Scale-up (Upscale)**: Processo de aumentar o número de réplicas ou recursos alocados a uma aplicação.

**Service**: Abstração do Kubernetes que define uma política de acesso lógico a um conjunto de Pods.

**Service Mesh**: Camada de infraestrutura dedicada que controla comunicação entre serviços em arquiteturas de microsserviços.

**Stabilization Window**: Período de tempo durante o qual o HPA observa métricas antes de tomar decisões de escalamento, prevenindo oscilações.

**Stateful Application**: Aplicação que mantém estado ou dados de sessão entre requisições, requerendo considerações especiais para escalabilidade.

**Stateless Application**: Aplicação que não mantém estado entre requisições, facilitando escalabilidade horizontal.

**Target**: Valor desejado para uma métrica, utilizado pelo HPA para calcular número necessário de réplicas.

**Threshold**: Valor limite que, quando ultrapassado, aciona ações de escalamento.

**Throughput**: Quantidade de trabalho processado em um determinado período de tempo, frequentemente medido em requisições por segundo.

**Upscale**: Sinônimo de scale-up, processo de aumentar capacidade.

**Utilization**: Percentual de uso de um recurso em relação ao valor configurado como request.

**Vertical Scaling**: Escalabilidade implementada através do aumento ou diminuição de recursos alocados a instâncias individuais.

**VPA (Vertical Pod Autoscaler)**: Componente do Kubernetes que ajusta automaticamente os resource requests e limits de containers baseado em utilização observada.

**Warm-up**: Período inicial após inicialização de uma aplicação durante o qual performance pode ser reduzida devido a caches frios ou inicializações pendentes.

**Workload**: Conjunto de recursos computacionais e aplicações executando em um cluster Kubernetes.
