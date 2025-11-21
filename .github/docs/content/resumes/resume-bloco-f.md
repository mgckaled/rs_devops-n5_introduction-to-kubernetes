# Resumo: Bloco F - Entendendo Mais Sobre Volumes

## Aula

### Aula 1. Volumes e StorageClass

Nesta aula, finalizamos a primeira parte do nosso módulo sobre Kubernetes, abordando conceitos de self-healing e escalabilidade. Vamos explorar volumes, essenciais para manter o estado das aplicações, especialmente bancos de dados, que não podem perder informações durante falhas. Discutimos o Storage Class, que define como os volumes são gerenciados no Kubernetes, e sua importância em ambientes de produção. Na próxima aula, vamos nos aprofundar nos volumes e sua hierarquia.

### Aula 2. PersistentVolume

Nesta aula, exploramos a relação entre Storage Class e volumes no Kubernetes. Discutimos os dois tipos de volume: efêmero, que não persiste após o ciclo de vida do container, e persistente, que reserva espaço e se comunica com o Storage Class. Abordamos a importância de usar volumes persistentes para garantir a integridade dos dados, especialmente em aplicações que precisam de armazenamento duradouro.

### Aula 3. PersistentVolumeClaim

Na aula de hoje, vamos explorar a relação entre volumes e aplicações no Kubernetes. Discutimos como o Storage Class interage com o provisionador e como os volumes reservam espaço no cluster. Aprendemos que a aplicação não se conecta diretamente ao volume, mas sim ao Persistent Volume Claim (PVC), que faz o requerimento do espaço. Vamos criar, na prática, um Storage Class, um volume e um PVC, e testar tudo isso com nossa aplicação.

### Aula 4. Criando o StorageClass

Nesta aula, vamos focar na parte prática da configuração do Kubernetes, especificamente na criação de um StorageClass. Começamos deletando um arquivo anterior e criando um novo na raiz do K8s. Abordamos a estrutura do YAML, incluindo propriedades como provisioner e Reclaim Policy. Demonstrei como aplicar a configuração e verificar os StorageClasses criados. Na próxima aula, vamos avançar para a criação do Persistent Volume.

### Aula 5. Reservando o espaço com o PV

Nesta aula, vamos criar um volume persistente no Kubernetes, utilizando um arquivo YAML chamado pv.yaml. Abordamos a estrutura do YAML, incluindo a versão da API, tipo de recurso e metadados. Discutimos a capacidade do volume, modos de acesso e políticas de recuperação. O foco foi no modo ReadWriteOnce, que permite acesso exclusivo a um nó. Finalizamos com a criação do volume e a verificação de suas propriedades. Agora, vamos avançar para o Persistent Volume Claim (PVC) para integrar isso à nossa aplicação.

### Aula 6. Criando o PVC

Nesta aula, vamos criar um Persistent Volume Claim (PVC) que faz parte do contexto da aplicação. Começamos definindo a estrutura do arquivo PVC, que é semelhante ao Persistent Volume (PV). Discutimos a capacidade de armazenamento e como o PVC requisita um valor específico, além de abordar a importância do Storage Class e dos access modes. Finalizamos com a criação do PVC e a verificação do seu status, que está pendente, aguardando um consumidor. Na próxima aula, vamos explorar quem realmente utilizará esse volume.

### Aula 7. Associando o PVC na nossa aplicação

Na aula de hoje, exploramos como interagir com containers no Kubernetes, utilizando comandos como kubectl exec para acessar arquivos dentro dos pods. Discutimos a importância de volumes persistentes, especialmente em aplicações que precisam manter dados, como uploads. Demonstrei como configurar um volume no deployment e como ele se comporta ao deletar pods. Ao final, enfatizei que, para aplicações que requerem persistência, o uso de volumes é essencial. Na próxima aula, faremos uma revisão do que aprendemos.

### Aula 8. Resumo sobre volumes

Finalizamos nossa aula sobre volumes, que é a parte mais básica do Kubernetes. Discutimos ConfigMap, Secrets, Services, HPA, Deployment, ReplicaSet e PVC, além das probes no Deployment. No próximo módulo, vamos explorar a observabilidade em microserviços no Kubernetes, abordando métricas, logs e alarmes. Depois, avançaremos para tópicos mais complexos, como Storage Class, Kubernetes em nuvem e Helm.
