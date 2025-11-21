# Resumo: Bloco E - Probes e Self Healing

## Aula

### Aula 1. O que são Probes e Self Healing

Nesta aula, finalizei o conteúdo sobre HPA e mergulhei no tema das probes e do self-healing no Kubernetes. Expliquei as três probes principais: Startup, Readiness e Liveness. Cada uma tem um papel crucial na verificação do estado da aplicação, desde garantir que o container subiu até monitorar sua saúde ao longo do tempo. Também discuti a importância de rotas específicas para testar dependências externas. Na próxima aula, vamos configurar essas rotas e probes.

### Aula 2. Configurando Rotas na Aplicação

Nesta aula, exploramos a criação de rotas na aplicação, focando na estrutura do controlador de saúde. Organizei o código em uma nova pasta chamada Health, onde criei o HealthController e o HealthService. Implementamos duas rotas: /health e /ready, que são essenciais para verificar a vivacidade e prontidão da aplicação. Também mencionei a biblioteca Terminus para testar dependências externas. Finalizamos com o build e push da imagem Docker, prontos para integrar com o Kubernetes.

### Aula 3. Startup

Nesta aula, exploramos a configuração do Startup Probe no Kubernetes. Discutimos como definir probes para monitorar a saúde da aplicação, utilizando rotas específicas para checagens. Abordamos a importância de configurar parâmetros como falhas toleradas, tempo de timeout e períodos de verificação. Também realizamos um teste prático, onde enfrentamos um erro 404 devido a uma rota inexistente, demonstrando o mecanismo de auto-recuperação do Kubernetes. Na próxima aula, vamos falar sobre o Readiness Probe.

### Aula 4. Readiness

Nesta aula, exploramos a configuração do readiness probe, que é fundamental para monitorar a prontidão da nossa aplicação. Aprendemos a definir rotas e parâmetros, como tempo de espera e thresholds. Também simulamos cenários reais, como tempos altos de bootstrap e erros aleatórios, para entender como esses fatores afetam a saúde da aplicação. Finalizamos com a construção do Docker e deixamos para a próxima aula a discussão sobre o liveness probe.

### Aula 5. Liveness

Nesta aula, exploramos a nova tag V8 e testamos cenários com o Liveness Probe. Discutimos como configurar parâmetros como falhas e sucessos, além de implementar o "initial delay seconds" para evitar falsos positivos. Testamos a aplicação em diferentes versões, observando como o Startup Probe pode causar reinicializações se a aplicação demorar a subir. Também abordamos a importância de monitorar a saúde da aplicação em produção, especialmente em situações de instabilidade. Na próxima aula, vamos aprofundar em comandos e scripts para probes.

### Aula 6. Refatorando a Aplicação e Entendendo Mais Sobre o Command

Nesta aula, analisamos uma aplicação instável que estava enfrentando múltiplos restarts. Discutimos a importância de corrigir problemas, como remover timeouts desnecessários e ajustar a estrutura do health service. Também falamos sobre a configuração de probes no Kubernetes, destacando a flexibilidade de usar apenas uma delas. Por fim, mencionei a possibilidade de executar scripts para verificações específicas. Nas próximas aulas, vamos aprofundar no tema de self-healing e explorar volumes e aplicações com estado.

### Aula 7. Garantindo prontidão da aplicação

Nesta aula, exploramos a configuração das probes: Startup, Liveness e Readiness, essenciais para garantir a saúde das aplicações em Kubernetes. Discutimos como essas probes ajudam na recuperação automática de falhas, evitando que containers problemáticos recebam tráfego. O Startup Probe impede que um pod inicie se a aplicação não estiver pronta, enquanto o Readiness verifica se está apta a receber requisições. O Liveness, por sua vez, monitora a saúde contínua do pod.

### Aula 8. Alternativas na camada da aplicação

Nesta aula, exploramos como evitar problemas com containers defeituosos, discutindo o impacto local e global que isso pode causar. Abordamos práticas como o Circuit Breaker, que ajuda a mitigar o "blast radius" de falhas, e o Fault Injection, que permite testar a resiliência da aplicação em cenários extremos. Essas técnicas são essenciais para construir aplicações mais robustas e seguras. Finalizamos com um teaser sobre o próximo módulo, que tratará de volumes em Kubernetes
