# Resumos: Bloco A - Conhecendo o Kubernets

## Aulas

### Aula 1. O que é Kubernetes

O novo módulo aborda o Kubernetes, começando com uma abordagem teórica antes de entrar em prática. O Kubernetes é um orquestrador de containers agnóstico a provedores de nuvem e tecnologias de container. Nasceu do projeto Borg do Google e é mantido pela CNCF. Automatiza a implantação de aplicações em containers de forma declarativa, facilitando a gestão de aplicações em escala. Porém, sua complexidade exige avaliar se é adequado para cada contexto.

### Aula 2. Contexto inicial e problema

Nesta aula, discuti sobre a execução de aplicações em contêineres e os desafios que surgem nesse cenário. Abordei a importância do Kubernetes para resolver problemas como falhas na execução, necessidade de múltiplos contêineres executando a mesma aplicação, fluxo elástico e controle de recursos. Destaquei a complexidade de lidar com esses desafios manualmente e como o Kubernetes pode ajudar nesse aspecto. Na próxima aula, vamos aprofundar esses pontos e esboçar soluções práticas.

### Aula 3. Esboço do problema

Nesta aula, discutimos a importância de containerizar aplicações no Kubernetes, destacando a necessidade de recursos computacionais, automação e efemeridade. Exploramos a redundância através da replicação horizontal de containers e a possibilidade de escalabilidade elástica. Abordamos a gestão de recursos, como memória e CPU, e a importância de manter aplicações stateless, salientando a automação de escalabilidade com HPA. Por fim, mencionamos a arquitetura do Kubernetes e seus principais elementos.

### Aula 4. Principais componentes

Esta aula aborda a arquitetura de um cluster Kubernetes, destacando o control plane e os nós de trabalho. O control plane gerencia o estado e a configuração do cluster, enquanto os nós executamos os containers. O kube-scheduler agenda a execução dos containers nos nós, considerando os recursos disponíveis. O etcd armazena as informações de configuração do cluster. O Kubelet comunica com o control plane e o Kube-proxy facilita as comunicações de rede.

### Aula 5. Como executar um Cluster Kubernetes

Nesta aula, expliquei sobre a execução do Kubernetes, destacando que é open source e portátil, podendo ser executado em qualquer lugar. Abordei as duas principais formas de execução: gerenciada, com custo para o gerenciamento em provedores de nuvem, e não gerenciada, onde a equipe cuida do gerenciamento. Também mencionei a possibilidade de execução em máquinas locais para estudo. Destaquei a importância de ferramentas para interagir com o cluster Kubernetes e a configuração mínima do ambiente na próxima aula.

### Aula 6. Configurando Ambiente

Nesta aula, expliquei sobre a importância do kubectl como uma CLI essencial para interagir com o Kubernetes, além de ferramentas como Lens e KNINS para facilitar a interface com o cluster. Destaquei o uso do kind para criar um ambiente local do Kubernetes, rodando o cluster em containers Docker. Também mencionei a instalação dessas ferramentas e a flexibilidade de escolha entre diferentes opções, como minikube e k3s. Ao final, mencionei o k9s como uma ferramenta adicional para estudos.

### Aula 7. Configurando e criando nosso primeiro Cluster

Nesta aula, foi demonstrado como criar um cluster Kubernetes utilizando o Kind. Foi explicado o processo de criação do cluster, a instalação de pré-requisitos como Docker e Go, e a execução do comando kind create cluster. Foi ressaltada a importância de separar o control plane dos workloads, mostrando que o cluster criado inicialmente possui apenas o control plane. Foi mencionado o uso do Lens para visualizar o cluster e a necessidade de ajustes para incluir os worker nodes. O vídeo termina com a promessa de abordar o gerenciamento de múltiplos nós na próxima aula.

### Aula 8. Configurando Cluster com múltiplos nós

Nesta aula, foi criado um cluster com um control plane e um nó de trabalho. O control plane gerencia o cluster, enquanto o nó de trabalho executa as cargas de trabalho. Foi mostrado como criar um novo cluster usando YAMLs do Kubernetes com o Kind. Foram abordados conceitos como redundância de control plane e nós de trabalho para garantir resiliência. A prática envolveu deletar e criar clusters com diferentes configurações
