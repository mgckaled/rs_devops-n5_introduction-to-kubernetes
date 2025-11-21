# Resumos: Bloco B - Orquestrando Containers

## Aulas

### Aula 1. Subindo o nosso primeiro Container no K8S

Nesta aula, exploramos a execução de um cluster Kubernetes com múltiplos control planes e workers. Focamos no objeto pod, a menor unidade hierárquica do Kubernetes, onde as aplicações são executadas. Utilizamos o Kubernetes para rodar um container nginx de forma imperativa, destacando a importância de versionamento e boas práticas. Discutimos a diferença entre abordagens imperativas e declarativas, e a necessidade de controladores para garantir a estabilidade das aplicações.

### Aula 2. Valor dos manifestos declarativos e namespaces

Nesta aula, abordo a criação de um namespace no Kubernetes para melhor organização das aplicações, a criação de um arquivo de configuração YAML para um pod com o container Nginx, a definição de recursos de CPU e memória no arquivo YAML, e a aplicação desse arquivo no cluster utilizando o comando `kubectl apply`. Também destaco a importância de definir namespaces para uma melhor governança das aplicações e a necessidade de especificar recursos de CPU e memória para os containers visando uma boa prática de uso de recursos.

### Aula 3. Acessando container dentro do cluster

Nesta aula, foi abordado o conceito de POD no Kubernetes e a importância da interface de rede do POD para acessar aplicações dentro do cluster. Foi demonstrado o uso do PORT FORWARD para acessar um POD via interface gráfica e via linha de comando. Também foi mostrado como verificar recursos e métricas do POD. Problemas comuns foram mencionados, a serem abordados em aulas futuras. O vídeo encerrou com a promessa de explorar mais recursos do Kubernetes.

### Aula 4. Problemas e próximos passos

Nesta aula, abordo a importância da redundância ao executar containers em um cluster Kubernetes. Destaco a necessidade de ter vários pods em execução para garantir disponibilidade, explicando que os pods são efêmeros e descartáveis. Também menciono a diferença de comportamento ao deletar um pod e introduzo a ideia de controladores, como replica set e deployment, que garantem o número correto de réplicas em execução. Exploro a importância desses conceitos para a próxima aula.

### Aula 5. Criando um ReplicaSet

Nesta aula, exploramos a efemeridade dos pods no Kubernetes e a importância da replicação para garantir a resiliência das aplicações stateless. Introduzimos o ReplicaSet como um controlador para gerenciar múltiplos pods, demonstrando como criar e configurar um ReplicaSet no Kubernetes. Aprendemos sobre a necessidade de definir labels e seletores para o ReplicaSet funcionar corretamente. Ao final, destacamos a diferença entre pods controlados diretamente e por um ReplicaSet, enfatizando a importância da replicação para manter a disponibilidade das aplicações.

### Aula 6. Problemas de um ReplicaSet

Nesta aula, exploro a questão do ReplicaSet e por que não o utilizaremos diretamente no curso. Abordo a importância das tags ao fazer deploy de aplicações no Kubernetes, destacando a prática de associar a tag ao commit feito na aplicação. Demonstro na prática a troca de tags e como o Replica Set não suporta essa funcionalidade, exigindo a exclusão e recriação do objeto. Destaco a indisponibilidade gerada por essa abordagem e menciono a necessidade de mecanismos como o zero downtime deployment. Na próxima aula, exploraremos um novo objeto da API do Kubernetes.

### Aula 7. Criando um Deployment

Nesta aula, foi abordado o conceito de Deployment no Kubernetes, comparando-o com ReplicaSet. Foi destacada a importância de definir o namespace ao aplicar manifestos YAML no cluster. O Deployment é responsável pela implantação e controle de versões, ao contrário do ReplicaSet. Foi demonstrado como criar um Deployment YAML, especificando réplicas, seletores, templates de pods e recursos. Por fim, foi exemplificado o processo de atualização de versões e o controle de disponibilidade com o Deployment e ReplicaSet.

### Aula 8. Acessando Pods

Esta aula aborda a importância do acesso no Kubernetes, explicando a variação na quantidade de pods e a necessidade de um componente de interface de rede, o Service. Mostra como criar um Service.yaml, detalhando o Cluster IP e os ports. Destaca a relação um para um entre Service e Deployment, e a importância do declarativo no Kubernetes. Aborda também a escala declarativa versus imperativa e a fonte da verdade no declarativo. Finaliza mencionando os próximos temas a serem abordados.
