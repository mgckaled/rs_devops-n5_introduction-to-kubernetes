# Resumos: Bloco B - Explorando Deployment e Cenários em uma Aplicação Real

## Aulas

### Aula 1. Conteinerizando a nossa aplicação

Durante a aula, exploramos conceitos como deployment, replica set, pod e service em um cluster Kubernetes. Utilizamos uma aplicação em NestJS como exemplo, criando um Dockerfile para containerizá-la. Demonstramos o processo de build, instalação de dependências, exposição de portas e execução do container. Discutimos a importância de otimizar o tamanho da imagem para facilitar o deploy. Na próxima aula, abordaremos o envio da imagem para o Docker Hub e o deploy em um cluster Kubernetes.

### Aula 2. Criando os Objetos do Kubernetes

Nesta aula, expliquei como enviar uma imagem Docker para um repositório remoto, destacando a importância de logar no container registry e taguear a imagem antes de fazer o push. Em seguida, foi abordado o processo de criar um arquivo deployment.yaml para implantar a aplicação no Kubernetes, incluindo a definição de réplicas, labels e recursos do container. Por fim, foi demonstrado como testar a aplicação no cluster Kubernetes usando port forward.

### Aula 3. Criando Service e explorando imagePullPolicy

Neste aula, foi abordado o processo de criação de um service no Kubernetes para facilitar o acesso à aplicação. Foi explicado o uso do arquivo service.yml para definir as configurações do service, como o tipo de API, o nome, o tipo de porta e o redirecionamento para o deployment. Também foi mencionada a importância de manter a imutabilidade das tags de imagem e o uso do image pool policy para garantir a consistência e rastreabilidade das versões. O instrutor exemplificou o processo de build, push e deploy de uma nova versão da aplicação, destacando a necessidade de cuidado ao lidar com as tags de imagem para evitar problemas de versionamento. O próximo passo seria explorar como lidar com possíveis problemas em uma aplicação e a importância de manter um controle adequado das versões no ambiente produtivo com o Kubernetes.

### Aula 4. Entendendo Problemas da Tag Latest

Nesta aula, foi abordado o conceito de image pool policy "always" e a prática de rollback em caso de problemas após um deploy. Foi demonstrado como verificar o histórico de deploys com o comando "kubectl rollout history" e como reverter para uma versão anterior com "kubectl rollout undo". Foi destacado um problema ao mutabilizar a tag da imagem, resultando em falhas no rollback. O instrutor propõe criar uma nova tag na próxima aula para lidar com essa situação.

### Aula 5. Criando Nova Tag e Controlando Rollback da Aplicação

Nesta aula, foram feitas alterações no image pool policy e na aplicação para evitar problemas de sobrescrita de tags. Foi mostrado o processo de build, push e deploy de uma nova versão da aplicação, com destaque para a estratégia de deploy cadenciada. Também foi abordado o conceito de rollback, mostrando como voltar para uma versão anterior em caso de problemas. Foi ressaltada a importância de manter o estado do cluster declarativo para evitar surpresas.

### Aula 6. Trabalhando Com Estratégias De Deploy

Nesta aula, exploramos estratégias de deploy, como aumentar o número de réplicas, virar de uma tag para outra e configurar estratégias de deploy no Kubernetes. A estratégia padrão é o rolling update, que permite controlar o número máximo de réplicas adicionais e indisponíveis durante a atualização. É possível configurar as propriedades maxSurge e maxUnavailable para otimizar o processo. Também foi mencionado o Type Recreate como outra estratégia a ser explorada.

### Aula 7. Entendendo o Recreate

Nesta aula, exploramos a estratégia de recreate no Kubernetes, que provoca indisponibilidade ao reiniciar todas as réplicas. Essa estratégia não é recomendada para aplicações em produção devido à interrupção do serviço. Discutimos também outras estratégias avançadas, como blue-green e canary deployment. Na próxima aula, abordaremos config map, env e secrets no Kubernetes. Esses tópicos mais avançados serão explorados ao longo do módulo.

### Aula 8. Explorando Variável de Ambiente na Aplicação

Nesta aula, exploramos a camada de configuração no Kubernetes, abordando ConfigMap e Secret. Variáveis de ambiente são essenciais para definir configurações como ambiente, URLs, tokens, etc. Criamos um arquivo .env com variáveis e aprendemos a carregá-lo em uma aplicação Node.js com NestJS. Para garantir segurança, não comitamos o .env no Git. Na próxima aula, configuraremos o .env no ConfigMap do Kubernetes para uso em tempo de execução.

### Aula 9. Entendendo sobre o ConfigMap

Nesta aula, exploramos a criação e utilização do objeto ConfigMap no Kubernetes. Iniciamos com a criação do arquivo .env e ajustes no C++ e ML. Em seguida, abordamos a importância de ignorar o .env no versionamento e no build local da imagem. Alteramos o deployment.yaml para RollingUpdate, configuramos réplicas e aplicamos as mudanças. Criamos o ConfigMap app.ts com a variável appname e injetamos no deployment para acessar a informação no pod. Finalizamos com a verificação do funcionamento da injeção de variável no pod.

### Aula 10. Explorando o objeto Secret

Nesta aula, exploramos o conceito de Secrets no Kubernetes, que são utilizados para armazenar informações sensíveis, como chaves de API, tokens, URLs de bancos de dados, entre outros. Diferentemente do ConfigMap, que lida com informações não sensíveis, as Secrets são recomendadas para dados sigilosos. Mostramos como criar e utilizar Secrets, destacando a importância de armazenar os valores em base64. Demonstramos como integrar as Secrets na aplicação e como lidar com múltiplas variáveis e Secrets no Deployment. Ao final, ressaltamos a importância de controlar e passar os valores de acordo com o ambiente.

### Aula 11. Melhorando Gerenciamento de Envs

Nesta aula, foi abordada a escalabilidade de manter e passar informações no Kubernetes, utilizando env e env from para injetar variáveis. Foi destacada a importância de garantir que as variáveis injetadas correspondam às esperadas pela aplicação. Também foi mencionado o uso do Vault para gerenciar segredos de forma externa ao Kubernetes. Ressaltamos a importância de revisar as variáveis e como elas são referenciadas na aplicação ao trabalhar com env.from.
