# Orquestração com Kubernetes

## Sobre

Repositório pessoal de registro, referência e suporte para fins de aprendizado, consulta e acompanhamento da disciplina de Orquestração - Kubernets (Nível 5), da foramção DevOps, desenvolvido pela Rocketseat.

## Conteúdo Teórico

### Bloco A: Fundamentos de Kubernetes

O Bloco A apresenta os conceitos fundamentais do Kubernetes (K8s), uma plataforma open source de orquestração de containers desenvolvida inicialmente pelo Google e mantida pela Cloud Native Computing Foundation (CNCF). Este módulo introduz o Kubernetes como solução para os desafios comuns da execução de aplicações containerizadas em escala, incluindo auto-recuperação de falhas, replicação horizontal, escalabilidade elástica, gestão de recursos e coordenação de múltiplas aplicações. O conteúdo aborda desde a origem do projeto no sistema Borg do Google até suas características principais como portabilidade, declaratividade e extensibilidade, estabelecendo uma base sólida para compreender quando e por que utilizar Kubernetes em ambientes de produção.

A arquitetura do Kubernetes é detalhada através da separação entre Control Plane e Nodes de trabalho, explicando o papel de cada componente essencial: API Server como ponto central de comunicação, etcd para armazenamento distribuído de estado, Scheduler para alocação de recursos, Controller Manager para manutenção do estado desejado, Kubelet como agente nos nodes, Kube-proxy para gerenciamento de rede, e Container Runtime para execução dos containers. O módulo também apresenta as diferentes formas de execução do Kubernetes, desde clusters gerenciados em provedores de nuvem até ambientes locais para desenvolvimento usando ferramentas como Kind, Minikube e k3s, incluindo guias práticos para configuração de ambiente e criação dos primeiros clusters com múltiplos nós, preparando o profissional para trabalhar tanto em cenários de estudo quanto em implementações de produção.

>[!NOTE]
> [Kubernetes: Fundamentos de Orquestração de Containers](./.github/docs/content/bloco-a.md)
>
> [Apresentação de Slides](./.github/docs/content/presentations/bloco-a.pdf)

### Bloco B: Orquestrando Containers com Pods, ReplicaSets, Deployments e Services

O Bloco B mergulha nos objetos fundamentais do Kubernetes para orquestração de containers, apresentando a hierarquia de recursos desde Pods até Services. O módulo explora profundamente a transição da abordagem imperativa para a declarativa, demonstrando como manifestos YAML permitem gerenciamento versionável e reproduzível através de GitOps. São abordados conceitos essenciais como Pods (unidade mínima efêmera de deployment), ReplicaSets (controladores que garantem redundância através de réplicas), Deployments (gerenciamento de atualizações com rolling updates e rollback), e Services (abstração de rede para exposição estável de aplicações). O conteúdo detalha a gestão de recursos computacionais (requests e limits), uso de namespaces para organização e isolamento, e a importância do versionamento adequado de imagens Docker.

A segunda parte do módulo foca em estratégias avançadas de deployment, explicando detalhadamente o funcionamento do RollingUpdate (atualização gradual sem downtime) e Recreate (substituição completa com interrupção), incluindo parâmetros críticos como maxSurge e maxUnavailable para controle fino do processo. São apresentados os diferentes tipos de Services (ClusterIP para comunicação interna, NodePort para acesso externo simples, LoadBalancer para produção em nuvem), mecanismos de descoberta de serviços via DNS e variáveis de ambiente, e a relação entre Deployments e Services através de labels e selectors. O módulo culmina com a apresentação do ciclo completo de deployment, desde a criação inicial até atualizações com zero downtime, incluindo boas práticas essenciais como sempre usar controladores ao invés de Pods standalone, especificar requests e limits, usar tags específicas de imagens, e manter manifestos versionados para garantir rastreabilidade e possibilitar rollbacks seguros.

>[!NOTE]
> [Kubernetes: Orquestrando Containers com Pods, ReplicaSets, Deployments e Services](./.github/docs/content/bloco-b.md)

### Bloco C: Deployment e Configuração de Aplicações em Produção

O Bloco C aprofunda o ciclo completo de deployment de aplicações containerizadas em Kubernetes, desde a containerização até a gestão de configurações em ambientes produtivos. O módulo inicia com a construção de imagens Docker otimizadas utilizando multi-stage builds para reduzir o tamanho final e melhorar a segurança, seguido pelo workflow completo de Container Registry (autenticação, push de imagens, e versionamento semântico). São apresentadas estratégias críticas de versionamento de imagens, explorando a política imagePullPolicy (Always, IfNotPresent, Never) e os problemas associados ao uso da tag latest, incluindo demonstrações práticas de como rollback e revision history funcionam quando tags são mutáveis versus imutáveis. O conteúdo aborda profundamente a separação entre código e configuração através de ConfigMaps (para dados não-sensíveis como URLs, feature flags) e Secrets (para credenciais, tokens, chaves de API), demonstrando diferentes formas de injeção (env, envFrom, volumes) e integrações com sistemas externos de gestão de segredos como HashiCorp Vault.

A segunda parte do módulo concentra-se em estratégias avançadas de deployment para ambientes de produção, detalhando implementações de Blue-Green Deployment (dois ambientes completos com switch instantâneo via Service selector) e Canary Deployment (liberação gradual controlada por peso de tráfego). São explorados mecanismos de rollback automático e manual através do histórico de revisões (revisionHistoryLimit, rollout history, rollout undo), incluindo cenários de falha e recuperação. O módulo culmina com a apresentação de ferramentas e práticas para gestão multi-ambiente, abordando Kustomize para overlays declarativos (base + patches por ambiente) e Helm para templating parametrizado com values files, além de boas práticas essenciais como nunca usar imagePullPolicy Always com tag latest, sempre versionar manifestos em Git, implementar health checks (liveness, readiness, startup probes), configurar resource limits adequados, e utilizar ferramentas de observabilidade para monitorar deployments em tempo real, preparando o profissional para operar aplicações Kubernetes em cenários produtivos complexos com alta disponibilidade e zero downtime.

>[!NOTE]
> [Kubernetes: Orquestrando Containers com Pods, ReplicaSets, Deployments e Services](./.github/docs/content/bloco-c.md)

### Bloco D - Conhecendo o HPA

>[!NOTE]
> [BlocoD](./.github/docs/content/bloco-d.md)

### Bloco E - Probes e Self Healing

>[!NOTE]
> [BlocoE](./.github/docs/content/bloco-e.md)

### Bloco F - Entendendo Mais Sobre Volumes

>[!NOTE]
> [BlocoF](./.github/docs/content/bloco-f.md)
