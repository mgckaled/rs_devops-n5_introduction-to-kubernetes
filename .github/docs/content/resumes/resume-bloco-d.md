# Resumo: Bloco D - Conhecendo o HPA

## Aulas

### Aula 1. O que é Escala

Nesta aula, expliquei sobre a importância da escala de uma aplicação em um cluster Kubernetes, destacando a necessidade de adaptar a aplicação a diferentes cenários de tráfego. Abordei a resiliência da aplicação em situações de alto tráfego, a importância da autoescala para lidar com picos de acesso e a realização de testes de carga e estresse. Na próxima aula, abordarei os tipos de escala e como implementá-los na prática.

### Aula 2. Conhecendo a escala vertical

Esta aula aborda a escala vertical de aplicações, explicando que consiste em aumentar o tamanho da máquina para lidar com mais recursos computacionais, como CPU, memória RAM e armazenamento. Destaca que a escala vertical é indicada para casos de grande escala com poucas máquinas, mas pode gerar problemas de redundância e downtime. Também menciona a importância do downsize para evitar ociosidade de hardware. No Kubernetes, é possível utilizar o Vertical Positive Outscale (VPA) para escala vertical.

### Aula 3. Explorando a escala horizontal

Nesta aula, expliquei sobre a escala horizontal em comparação com a escala vertical no contexto de servidores. Abordei a replicação de infraestrutura, distribuição de cargas de trabalho, redundância, tolerância a falhas e a importância de recursos como CPU e memória. Destaquei a autoescala baseada em condições e eventos, além de mencionar o KEDA para escalonamento baseado em eventos. Também mencionei desafios com consistência de dados em aplicações stateful e apresentei o HPA como componente essencial no Kubernetes.

### Aula 4. Como o Metrics-Server funciona

O Metrics Server é um componente do Kubernetes que captura métricas em quase tempo real, essenciais para o funcionamento do Horizontal Pod Autoscaler (HPA). Sem o Metrics Server, o HPA não pode operar. O Metrics Server monitora as métricas da aplicação e do cluster, permitindo a configuração do autoescalamento. Na próxima aula, vamos instalar e configurar o Metrics Server no cluster para habilitar o autoescalamento.

### Aula 5. Adicionando o Metrics-Server no nosso Cluster

Commit

Nesta aula, foi abordada a instalação do Matrix Server em um cluster Kubernetes existente. Foi destacada a importância de seguir a documentação e foram mostrados os recursos criados durante a instalação, como conta de serviço, permissões RBAC e deployment. Problemas com certificados foram resolvidos desabilitando a verificação do certificado. Foi demonstrado como baixar e aplicar um arquivo YAML, além de adicionar tags de configuração recomendadas. Por fim, foi mencionada a configuração futura do HPA e a visualização de métricas.

### Aula 6. Entendendo os Principais Triggers

Nesta aula, exploramos o Horizontal Pod Autoscaler (HPA) no Kubernetes, um recurso que permite a escalabilidade automática de pods com base em métricas de uso, como CPU e memória. Discutimos como o HPA monitora a utilização dos recursos e ajusta o número de réplicas conforme necessário, garantindo alta disponibilidade. Também abordamos a importância de definir limites máximos para evitar problemas de escalabilidade. Na próxima aula, vamos criar nosso primeiro manifesto HPA e aplicá-lo no cluster.

### Aula 7. Explorando a V1 do HPA

Nesta aula, vamos criar nosso primeiro manifesto de Horizontal Pod Autoscaler (HPA) no Kubernetes, focando na versão v1 e, em seguida, na v2. Abordaremos como configurar a autoescala de uma aplicação, definindo o número mínimo e máximo de réplicas, além de configurar a utilização de CPU como gatilho para escalonamento. Também discutiremos a importância de escolher valores adequados para evitar sobrecarga ou ociosidade. Por fim, faremos a transição para a v2, que oferece mais opções.

### Aula 8. Criando o HPA Utilizando a V2

Nesta aula, exploramos a versão 2 do Horizontal Pod Autoscaler (HPA) e suas configurações. Começamos criando um novo arquivo HPA, definindo a versão da API e especificando as métricas de CPU e memória. Aprendemos a personalizar a escala com base na utilização média, ajustando os limites de réplicas. Observamos como o HPA reage a altas utilizações, escalando automaticamente os pods. Na próxima aula, vamos abordar testes de carga e configurações adicionais para escalonamento.

### Aula 9. Estressando a Nossa Aplicação

Na aula de hoje, exploramos como simular um cenário de alto tráfego utilizando o FortIO para realizar testes de estresse em uma aplicação no Kubernetes. Aprendemos a configurar e executar o FortIO para gerar carga, monitorando o comportamento da aplicação e a escalabilidade do HPA. Observamos o consumo de recursos e como a aplicação se comporta sob pressão. Para a próxima aula, planejamos adicionar complexidade à aplicação e repetir os testes.

### Aula 10. Explorando mais cenários de estresse

Nesta aula, exploramos um exemplo mais avançado de como nossa aplicação se comporta sob carga. Mantivemos o número de réplicas e recursos, mas observamos a redução das réplicas devido ao baixo uso de CPU. Implementamos um processo de escrita em arquivo usando streams no Node.js, o que gerou alto consumo de CPU. Realizamos testes de estresse para entender a performance e discutimos a importância de ajustar réplicas e recursos conforme a demanda. Na próxima aula, faremos mais testes para analisar os resultados.

### Aula 11. Alterando Recursos e Réplicas da Aplicação

Nessa aula, exploramos como escalar nossa infraestrutura e realizar um teste de estresse para aumentar o número de requisições. Começamos configurando o HPA com réplicas e ajustando os limites de CPU. Após aplicar as mudanças, rodamos o teste e observamos um aumento significativo nas requisições, quase quatro vezes mais, com um tempo de resposta reduzido. Também discutimos a importância de calibrar o trigger de utilização e como a estabilização dos pods pode ser configurada na versão 2 do HPA, que veremos na próxima aula.

### Aula 12. Definindo tempo de reação para escalonar a quantidade de réplicas

Nesta aula, exploramos a janela de estabilização do HPA (Horizontal Pod Autoscaler) para gerenciar a escalabilidade de aplicações. Discutimos como configurar métricas, como CPU e memória, e a importância de ter pelo menos uma métrica para que o HPA funcione corretamente. Abordamos também as políticas de scale up e scale down, permitindo um controle mais refinado sobre a quantidade de réplicas. Finalizamos com uma demonstração prática, observando como o HPA reage a picos de acesso e ajusta as réplicas de forma eficiente.
