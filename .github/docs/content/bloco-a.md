<!-- markdownlint-disable -->

# Kubernetes: Fundamentos de Orquestração de Containers

## 1. Resumo Executivo

O Kubernetes, também conhecido como K8s, é uma plataforma open source de orquestração de containers que automatiza a implantação, o dimensionamento e a gestão de aplicações containerizadas. Desenvolvido inicialmente pelo Google e atualmente mantido pela Cloud Native Computing Foundation (CNCF), o Kubernetes tornou-se o padrão de facto para orquestração de containers em ambientes de produção. Este documento apresenta os conceitos fundamentais, arquitetura e componentes essenciais do Kubernetes, fornecendo uma base sólida para profissionais que desejam compreender e implementar soluções de orquestração em escala.

## 2. Introdução e Conceitos

### 2.1. O que é Kubernetes

O Kubernetes é um orquestrador de containers que resolve problemas complexos de gestão de aplicações distribuídas. Sua origem remonta ao projeto Borg do Google, sistema interno utilizado por mais de uma década para gerenciar bilhões de containers. Em 2014, o Google disponibilizou o Kubernetes como projeto open source, e desde então a CNCF assumiu sua manutenção e evolução.

O termo "Kubernetes" deriva do grego e significa "timoneiro" ou "piloto", refletindo sua função de guiar e gerenciar cargas de trabalho containerizadas. A abreviação K8s representa as oito letras entre "K" e "s" no nome completo.

### 2.2. Características Principais

O Kubernetes se destaca por ser:

- **Agnóstico a provedores**: Funciona em qualquer ambiente, seja on-premises, cloud pública ou híbrida
- **Declarativo**: A configuração desejada é especificada e o sistema trabalha para manter esse estado
- **Escalável**: Projetado para gerenciar aplicações desde pequena até grande escala
- **Extensível**: Permite personalização através de APIs e plugins
- **Portável**: Open source e compatível com múltiplas tecnologias de containerização

### 2.3. Cenário e Problemas Resolvidos

A execução de aplicações em containers apresenta desafios específicos que o Kubernetes resolve:

#### Problemas Comuns

1. **Falhas de Execução**: O que acontece quando um container falha? Como garantir que a aplicação continue disponível?

2. **Múltiplas Réplicas**: Como gerenciar vários containers executando a mesma aplicação para distribuir carga e garantir redundância?

3. **Elasticidade**: Como escalar automaticamente conforme a demanda, aumentando ou reduzindo recursos dinamicamente?

4. **Gerenciamento em Escala**: Como administrar dezenas ou centenas de aplicações, cada uma com múltiplos containers?

5. **Controle de Recursos**: Como limitar e garantir o uso adequado de CPU, memória e outros recursos computacionais?

#### Soluções do Kubernetes

O Kubernetes aborda esses desafios através de:

- **Auto-recuperação (Self-healing)**: Detecta e substitui containers com falha automaticamente
- **Replicação Horizontal**: Mantém o número desejado de réplicas em execução
- **Escalabilidade Automática**: Ajusta recursos baseado em métricas através do HPA (Horizontal Pod Autoscaler)
- **Gestão Declarativa**: Permite definir o estado desejado e o sistema mantém esse estado
- **Isolamento de Recursos**: Implementa limites e requisitos de recursos por container

### 2.4. Requisitos e Considerações

Para utilizar Kubernetes efetivamente, as aplicações devem:

- **Ser containerizadas**: Empacotadas em imagens de container (Docker, containerd, CRI-O)
- **Operar de forma stateless**: Preferencialmente sem dependência de estado local
- **Ser efêmeras**: Preparadas para serem criadas e destruídas conforme necessário
- **Expor health checks**: Fornecer endpoints para verificação de saúde

É importante avaliar se o Kubernetes é adequado para cada contexto. Sua complexidade pode ser excessiva para aplicações simples ou equipes pequenas sem necessidade de escala.

## 3. Arquitetura do Kubernetes

### 3.1. Visão Geral da Arquitetura

Um cluster Kubernetes é composto por dois tipos principais de componentes:

1. **Control Plane (Plano de Controle)**: Gerencia o estado global do cluster
2. **Nodes (Nós de Trabalho)**: Executam as cargas de trabalho (containers)

A arquitetura segue o padrão mestre-trabalhador, onde o Control Plane toma decisões sobre o cluster e os Nodes executam os containers conforme determinado.

### 3.2. Control Plane

O Control Plane é responsável por manter o estado desejado do cluster. Seus componentes principais são:

#### 3.2.1. API Server

O kube-apiserver é o componente central do Kubernetes. Todas as comunicações passam por ele:

- Expõe a API REST do Kubernetes
- Valida e processa requisições
- Atualiza objetos no etcd
- É o único componente que comunica diretamente com o etcd

#### 3.2.2. etcd

Banco de dados distribuído chave-valor que armazena:

- Toda a configuração do cluster
- Estado de todos os objetos Kubernetes
- Secrets e ConfigMaps
- Informações de rede e serviços

Características do etcd:

- Altamente disponível e consistente
- Utiliza algoritmo de consenso Raft
- Recomenda-se configurar em cluster (3, 5 ou 7 nós)
- Pode ser externo ou integrado ao Control Plane

#### 3.2.3. Scheduler (kube-scheduler)

Responsável por decidir em qual Node um Pod deve ser executado:

1. Observa novos Pods sem Node atribuído
2. Avalia recursos disponíveis em cada Node
3. Considera restrições e afinidades
4. Seleciona o Node mais adequado
5. Atualiza o Pod com a decisão

Fatores considerados na decisão:

- Recursos disponíveis (CPU, memória)
- Requisitos e limites do Pod
- Afinidades e anti-afinidades
- Taints e tolerations
- Volume requirements

#### 3.2.4. Controller Manager (kube-controller-manager)

Executa controladores que regulam o estado do cluster:

- **Node Controller**: Monitora e responde quando Nodes ficam indisponíveis
- **Replication Controller**: Mantém o número correto de Pods para cada ReplicaSet
- **Endpoints Controller**: Popula objetos Endpoints (conecta Services e Pods)
- **Service Account & Token Controllers**: Criam contas e tokens de acesso padrão

Os controladores seguem o padrão de reconciliação:

1. Observam o estado atual
2. Comparam com o estado desejado
3. Executam ações para convergir ao estado desejado

#### 3.2.5. Cloud Controller Manager (Opcional)

Integra com APIs de provedores de nuvem:

- Gerencia Nodes (cria/remove instâncias)
- Configura rotas de rede
- Provisiona Load Balancers
- Gerencia volumes de armazenamento

### 3.3. Nodes (Nós de Trabalho)

Os Nodes executam os containers e fornecem o ambiente de runtime. Componentes principais:

#### 3.3.1. Kubelet

Agente que executa em cada Node:

- Comunica-se com o API Server
- Garante que os containers estão em execução conforme especificado
- Monitora a saúde dos Pods através de probes
- Reporta o status do Node e dos Pods
- Gerencia o ciclo de vida dos containers

#### 3.3.2. Kube-proxy

Componente de rede que mantém regras de rede nos Nodes:

- Implementa o conceito de Services
- Configura regras de iptables ou IPVS
- Possibilita comunicação dentro e fora do cluster
- Gerencia balanceamento de carga entre Pods

#### 3.3.3. Container Runtime

Software responsável por executar containers:

- **containerd**: Runtime padrão, leve e eficiente
- **CRI-O**: Runtime otimizado para Kubernetes
- **Docker Engine**: Suportado via dockershim (deprecated)

Deve implementar a Container Runtime Interface (CRI).

### 3.4. Fluxo de Comunicação

Exemplo de fluxo quando um Deployment é criado:

1. Cliente executa `kubectl create deployment`
2. Requisição enviada ao API Server
3. API Server valida e armazena no etcd
4. Controller Manager detecta novo Deployment
5. Controller cria ReplicaSet necessário
6. ReplicaSet Controller cria Pods
7. Scheduler atribui Pods aos Nodes
8. Kubelet nos Nodes detecta novos Pods
9. Kubelet instrui Container Runtime a criar containers
10. Kube-proxy atualiza regras de rede se necessário

### 3.5. Alta Disponibilidade

Para produção, recomenda-se configurar redundância:

#### Control Plane

- Múltiplas instâncias do API Server (load balanced)
- Cluster etcd com 3, 5 ou 7 membros
- Múltiplas instâncias de Scheduler e Controller Manager (eleição de líder via Lease API)

#### Nodes

- Distribuição de Pods em múltiplos Nodes
- Anti-afinidades para separar réplicas
- Pod Disruption Budgets para manutenção segura

## 4. Formas de Execução

### 4.1. Clusters Gerenciados (Managed)

Provedores de nuvem oferecem Kubernetes como serviço:

#### Vantagens

- Control Plane gerenciado pelo provedor
- Atualizações e patches automatizados
- Integração nativa com serviços do provedor
- SLA garantido

#### Principais Ofertas

- **Amazon EKS** (Elastic Kubernetes Service)
- **Google GKE** (Google Kubernetes Engine)
- **Azure AKS** (Azure Kubernetes Service)
- **DigitalOcean Kubernetes**
- **IBM Cloud Kubernetes Service**

#### Considerações

- Custo do gerenciamento (além das VMs)
- Lock-in com provedor específico
- Menor controle sobre Control Plane

### 4.2. Clusters Não Gerenciados (Self-managed)

Instalação e gestão completa pela equipe:

#### Ferramentas de Instalação

- **kubeadm**: Ferramenta oficial para bootstrap de clusters
- **kops**: Automação para AWS e GCE
- **Kubespray**: Ansible playbooks para instalação

#### Vantagens

- Controle total sobre configuração
- Sem custos de gerenciamento
- Flexibilidade máxima

#### Desafios

- Responsabilidade total por atualizações
- Necessidade de expertise em infraestrutura
- Gerenciamento de alta disponibilidade
- Monitoramento e troubleshooting

### 4.3. Ambientes Locais para Desenvolvimento

Para estudo e desenvolvimento local:

#### Kind (Kubernetes in Docker)

```bash
# Criar cluster simples
kind create cluster

# Cluster com múltiplos nodes
kind create cluster --config kind-config.yaml
```

Exemplo de configuração com múltiplos nós:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

**Características**:

- Executa cluster em containers Docker
- Rápido para criar e destruir
- Ideal para testes e CI/CD
- Suporta múltiplos nós

#### Minikube

```bash
# Iniciar cluster
minikube start

# Com driver específico
minikube start --driver=docker
```

**Características**:

- VM ou container para cluster local
- Add-ons integrados (dashboard, ingress)
- Suporta diferentes drivers
- Ideal para desenvolvimento

#### k3s

```bash
# Instalação simples
curl -sfL https://get.k3s.io | sh -
```

**Características**:

- Kubernetes leve (40MB binary)
- Otimizado para IoT e Edge
- Produção-ready
- Baixo consumo de recursos

### 4.4. Ferramentas de Interação

#### kubectl

CLI oficial para interagir com clusters:

```bash
# Visualizar nós
kubectl get nodes

# Ver pods em todos namespaces
kubectl get pods --all-namespaces

# Aplicar configuração
kubectl apply -f deployment.yaml

# Verificar logs
kubectl logs pod-name

# Executar comando em pod
kubectl exec -it pod-name -- /bin/bash
```

#### Lens

- Interface gráfica completa
- Gerenciamento multi-cluster
- Visualização de recursos em tempo real
- Terminal integrado

#### K9s

- Interface de terminal interativa
- Navegação rápida entre recursos
- Visualização de logs e eventos
- Atalhos de teclado eficientes

#### kubecolor

- Adiciona cores à saída do kubectl
- Melhora legibilidade
- Drop-in replacement para kubectl

## 5. Configuração de Ambiente Local

### 5.1. Pré-requisitos

#### Sistema Operacional

- Linux, macOS ou Windows (com WSL2)
- 4GB RAM mínimo (8GB recomendado)
- 20GB espaço em disco

#### Docker

```bash
# Verificar instalação
docker --version

# Testar execução
docker run hello-world
```

### 5.2. Instalação do kubectl

#### Linux

```bash
# Download da versão mais recente
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Tornar executável
chmod +x kubectl

# Mover para PATH
sudo mv kubectl /usr/local/bin/

# Verificar instalação
kubectl version --client
```

#### macOS

```bash
# Via Homebrew
brew install kubectl

# Verificar
kubectl version --client
```

#### Windows

```powershell
# Via Chocolatey
choco install kubernetes-cli

# Verificar
kubectl version --client
```

### 5.3. Instalação do Kind

#### Linux/macOS

```bash
# Download
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64

# Tornar executável
chmod +x ./kind

# Mover para PATH
sudo mv ./kind /usr/local/bin/kind
```

#### Via Go

```bash
go install sigs.k8s.io/kind@latest
```

### 5.4. Criação do Primeiro Cluster

#### Cluster Simples

```bash
# Criar cluster com nome padrão
kind create cluster

# Verificar cluster
kubectl cluster-info --context kind-kind

# Listar nós
kubectl get nodes
```

Este comando cria um cluster com um único nó que atua como Control Plane.

#### Cluster com Múltiplos Nós

Criar arquivo `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

Executar criação:

```bash
# Deletar cluster anterior (se existir)
kind delete cluster

# Criar novo cluster com configuração
kind create cluster --config kind-config.yaml

# Verificar nós
kubectl get nodes
```

Saída esperada:

```text
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   1m    v1.27.0
kind-worker          Ready    <none>          1m    v1.27.0
kind-worker2         Ready    <none>          1m    v1.27.0
kind-worker3         Ready    <none>          1m    v1.27.0
```

#### Cluster com Control Plane Redundante

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
```

Este setup simula alta disponibilidade do Control Plane.

### 5.5. Configuração Avançada do Kind

#### Com Load Balancer e Port Mapping

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

#### Com Configurações de Rede Customizadas

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  disableDefaultCNI: false
nodes:
- role: control-plane
- role: worker
```

### 5.6. Gerenciamento de Contextos

```bash
# Listar contextos disponíveis
kubectl config get-contexts

# Mudar de contexto
kubectl config use-context kind-kind

# Ver configuração atual
kubectl config current-context

# Ver toda configuração
kubectl config view
```

### 5.7. Verificação do Cluster

```bash
# Informações do cluster
kubectl cluster-info

# Componentes do sistema
kubectl get pods -n kube-system

# Eventos do cluster
kubectl get events --all-namespaces

# Saúde dos componentes
kubectl get componentstatuses
```

## 6. Conclusões

O Kubernetes representa uma evolução significativa na forma como aplicações distribuídas são gerenciadas e operadas. Sua arquitetura robusta, baseada em componentes especializados trabalhando de forma coordenada, fornece as capacidades necessárias para orquestração de containers em escala.

### Principais Aprendizados

1. **Separação de Responsabilidades**: A arquitetura do Kubernetes separa claramente o Control Plane (gerenciamento) dos Nodes (execução), permitindo escalabilidade e manutenibilidade.

2. **Modelo Declarativo**: A abordagem declarativa simplifica a gestão de aplicações, permitindo que o sistema trabalhe continuamente para manter o estado desejado.

3. **Flexibilidade de Implantação**: Existem opções tanto para ambientes gerenciados quanto não gerenciados, permitindo escolher o nível de controle e responsabilidade adequado.

4. **Ecossistema Rico**: Ferramentas como kubectl, Kind, Lens e K9s facilitam a interação e gerenciamento de clusters em diferentes contextos.

5. **Complexidade Justificada**: Embora complexo, o Kubernetes resolve problemas reais de escala, resiliência e automação que seriam extremamente difíceis de gerenciar manualmente.

### Próximos Passos

Após compreender os fundamentos apresentados, os próximos tópicos a explorar incluem:

- **Objetos do Kubernetes**: Pods, Deployments, Services, ConfigMaps, Secrets
- **Rede no Kubernetes**: CNI plugins, Ingress, Network Policies
- **Armazenamento**: Persistent Volumes, Storage Classes
- **Configuração e Segurança**: RBAC, Service Accounts, Security Contexts
- **Observabilidade**: Logging, Monitoring, Tracing
- **CI/CD**: Integração com pipelines de deployment

### Recomendações

Para organizações considerando adoção do Kubernetes:

1. **Avaliar Necessidade**: Kubernetes adiciona complexidade; certifique-se de que os benefícios justificam o investimento.

2. **Começar Pequeno**: Inicie com aplicações não críticas para ganhar experiência.

3. **Investir em Capacitação**: A curva de aprendizado é significativa; treinamento adequado é essencial.

4. **Considerar Managed Services**: Para equipes pequenas ou sem expertise em infraestrutura, serviços gerenciados podem ser mais adequados.

5. **Planejar Alta Disponibilidade**: Em produção, sempre configure redundância no Control Plane e distribua workloads.

6. **Automatizar desde o Início**: Utilize GitOps e Infrastructure as Code para gerenciar configurações.

## 7. Referências Bibliográficas

### Documentação Oficial

- Kubernetes Documentation. "Kubernetes Components". Disponível em: https://kubernetes.io/docs/concepts/overview/components/. Acesso em: 2024.

- Kubernetes Documentation. "Cluster Architecture". Disponível em: https://kubernetes.io/docs/concepts/architecture/. Acesso em: 2024.

- Kubernetes Documentation. "Installing kubeadm". Disponível em: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/. Acesso em: 2024.

- Kubernetes Documentation. "Configure High Availability". Disponível em: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/. Acesso em: 2024.

### Projetos e Ferramentas

- Kind (Kubernetes in Docker). "Quick Start". Disponível em: https://kind.sigs.k8s.io/docs/user/quick-start/. Acesso em: 2024.

- etcd Documentation. "What is etcd". Disponível em: https://etcd.io/docs/. Acesso em: 2024.

- Cloud Native Computing Foundation (CNCF). "Kubernetes Project". Disponível em: https://www.cncf.io/projects/kubernetes/. Acesso em: 2024.

### Livros e Publicações

- Burns, Brendan; Beda, Joe; Hightower, Kelsey. "Kubernetes: Up and Running". O'Reilly Media, 2019.

- Luksa, Marko. "Kubernetes in Action". Manning Publications, 2018.

### Artigos e Whitepapers

- Google. "Borg, Omega, and Kubernetes: Lessons learned from three container-management systems over a decade". ACM Queue, 2016.

## 8. Apêndice

### Apêndice A: Comandos kubectl Essenciais

#### Informações do Cluster

```bash
# Versão do kubectl e cluster
kubectl version

# Informações do cluster
kubectl cluster-info

# Configuração do cluster
kubectl config view
```

#### Gerenciamento de Recursos

```bash
# Listar recursos
kubectl get <resource>
kubectl get pods
kubectl get deployments
kubectl get services

# Descrever recurso específico
kubectl describe <resource> <name>
kubectl describe pod my-pod

# Criar recurso de arquivo
kubectl apply -f config.yaml

# Deletar recurso
kubectl delete <resource> <name>
kubectl delete pod my-pod
```

#### Debugging e Logs

```bash
# Logs de um pod
kubectl logs <pod-name>

# Logs em tempo real
kubectl logs -f <pod-name>

# Executar comando em pod
kubectl exec -it <pod-name> -- <command>
kubectl exec -it my-pod -- /bin/bash

# Port forwarding
kubectl port-forward <pod-name> <local-port>:<pod-port>
```

#### Gerenciamento de Namespaces

```bash
# Listar namespaces
kubectl get namespaces

# Criar namespace
kubectl create namespace <name>

# Usar namespace específico
kubectl get pods -n <namespace>

# Configurar namespace padrão
kubectl config set-context --current --namespace=<namespace>
```

### Apêndice B: Estrutura de Manifesto YAML

#### Pod Básico

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

#### Deployment

```yaml
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
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Apêndice C: Arquitetura de Referência para Produção

#### Control Plane

- **Quantidade**: Mínimo 3 nós para alta disponibilidade
- **Recursos**: 2 vCPU, 4GB RAM por nó (mínimo)
- **Separação**: Control Plane isolado dos workloads via taints

#### etcd

- **Configuração**: Cluster dedicado com 3, 5 ou 7 membros
- **Recursos**: 2 vCPU, 8GB RAM, SSD para storage
- **Backup**: Snapshots automatizados frequentes

#### Worker Nodes

- **Quantidade**: Baseado na carga de trabalho
- **Recursos**: Variável conforme aplicações
- **Distribuição**: Múltiplas zonas de disponibilidade

#### Rede

- **CNI**: Calico, Cilium, ou Flannel
- **Ingress**: NGINX Ingress Controller ou Traefik
- **Service Mesh**: Istio ou Linkerd (opcional)

### Apêndice D: Troubleshooting Comum

#### Pod não inicia

```bash
# Verificar status
kubectl get pods

# Ver eventos
kubectl describe pod <pod-name>

# Verificar logs
kubectl logs <pod-name>

# Verificar configuração
kubectl get pod <pod-name> -o yaml
```

#### Node NotReady

```bash
# Verificar status do node
kubectl describe node <node-name>

# Verificar kubelet
systemctl status kubelet

# Logs do kubelet
journalctl -u kubelet -f
```

#### Problemas de Rede

```bash
# Testar conectividade entre pods
kubectl run test-pod --image=busybox -it --rm -- ping <pod-ip>

# Verificar DNS
kubectl run test-pod --image=busybox -it --rm -- nslookup kubernetes.default

# Verificar kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

### Apêndice E: Glossário e Termos Técnicos

**API Server**: Componente do Control Plane que expõe a API do Kubernetes e processa requisições REST.

**CRI (Container Runtime Interface)**: Interface padrão para integração de runtimes de container com o Kubernetes.

**CNI (Container Network Interface)**: Especificação para configuração de rede em containers Linux.

**Control Plane**: Conjunto de componentes que gerenciam o estado global do cluster Kubernetes.

**Declarativo**: Abordagem onde se especifica o estado desejado, e o sistema trabalha para alcançá-lo.

**etcd**: Banco de dados distribuído chave-valor utilizado para armazenar dados de configuração do Kubernetes.

**HPA (Horizontal Pod Autoscaler)**: Controlador que escala automaticamente o número de Pods baseado em métricas.

**Imperativo**: Abordagem onde se especifica comandos diretos para executar ações.

**K8s**: Abreviação de Kubernetes (K + 8 letras + s).

**kubelet**: Agente que executa em cada Node e garante que containers estejam rodando conforme especificado.

**kube-proxy**: Componente de rede que mantém regras de rede nos Nodes e implementa Services.

**Lease**: Objeto usado para coordenação e eleição de líder entre componentes.

**Manifesto**: Arquivo (geralmente YAML) que descreve recursos Kubernetes.

**Node**: Máquina (física ou virtual) que executa containers no cluster Kubernetes.

**Orquestração**: Automação da configuração, coordenação e gerenciamento de sistemas computacionais.

**Pod**: Menor unidade de deployment no Kubernetes, agrupa um ou mais containers.

**Reconciliação**: Processo contínuo de comparar estado atual com estado desejado e tomar ações corretivas.

**Scheduler**: Componente que decide em qual Node cada Pod deve executar.

**Self-healing**: Capacidade do sistema de detectar e recuperar-se automaticamente de falhas.

**Stateless**: Aplicação que não mantém estado persistente localmente, facilitando escalabilidade.

**Taint**: Marca aplicada a Nodes para repelir Pods, exceto aqueles com tolerations correspondentes.

**Workload**: Aplicação ou conjunto de aplicações executando no Kubernetes.

**YAML (YAML Ain't Markup Language)**: Formato de serialização de dados utilizado para manifestos Kubernetes.