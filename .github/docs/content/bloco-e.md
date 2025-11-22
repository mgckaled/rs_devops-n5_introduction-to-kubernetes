<!-- markdownlint-disable -->

# Bloco E - Probes e Self Healing no Kubernetes

## Resumo Executivo

Os mecanismos de probes e self-healing no Kubernetes representam capacidades fundamentais que transformam a plataforma de simples orquestrador de containers em sistema inteligente capaz de detectar, diagnosticar e remediar automaticamente problemas de saúde de aplicações. Este documento explora sistematicamente os três tipos de probes oferecidos pelo Kubernetes - Startup, Readiness e Liveness - cada um projetado para endereçar aspecto específico do ciclo de vida e saúde de containers, formando sistema abrangente de monitoramento e recuperação automática.

O self-healing no Kubernetes fundamenta-se na capacidade do kubelet de executar verificações periódicas sobre containers, interpretando resultados destas verificações para tomar decisões automatizadas sobre reinicialização de containers, remoção de Pods de load balancers, e gestão do tráfego de entrada. Esta automação elimina necessidade de intervenção manual para cenários comuns de falha, melhorando dramaticamente a disponibilidade de aplicações e reduzindo significativamente o Mean Time To Recovery (MTTR) em ambientes de produção.

Startup Probes endereçam especificamente o desafio de containers com tempos de inicialização prolongados, prevenindo que Liveness Probes terminem prematuramente containers que ainda estão no processo de bootstrap. Readiness Probes controlam quando containers devem receber tráfego de produção, permitindo que aplicações sinalizem que estão prontas para processar requisições apenas após conclusão de inicializações complexas, carregamento de dados, ou estabelecimento de conexões com dependências externas. Liveness Probes monitoram continuamente saúde de containers durante toda sua vida útil, detectando estados irrecuperáveis como deadlocks ou corrupção de memória que requerem reinicialização forçada.

A implementação efetiva de probes requer compreensão profunda das características específicas de cada aplicação, incluindo tempos típicos de inicialização, padrões de falha observados, dependências críticas, e tolerâncias aceitáveis para latência de detecção versus overhead de verificações. Configurações inadequadas podem resultar em falsos positivos que causam reinicializações desnecessárias, ou falsos negativos que permitem containers defeituosos continuem servindo tráfego. Este documento explora não apenas mecânicas técnicas de cada tipo de probe, mas também padrões de design, melhores práticas e estratégias complementares no nível de aplicação como Circuit Breakers e Fault Injection que juntos constroem arquiteturas verdadeiramente resilientes.

## 1. Introdução e Conceitos

### 1.1. Desafios de Saúde em Ambientes Distribuídos

Aplicações distribuídas executando em containers enfrentam modos de falha complexos que transcendem simples crashes de processo. Containers podem entrar em estados degenerados onde o processo principal continua executando mas a aplicação não consegue mais processar requisições devido a deadlocks, esgotamento de conexões de pool, corrupção de memória, ou perda de comunicação com dependências críticas. Em modelos tradicionais de operação, estes estados requerem intervenção manual para diagnóstico e recuperação, resultando em períodos prolongados de indisponibilidade.

O Kubernetes aborda este desafio através de sistema sofisticado de health checking que permite aplicações comunicarem seu estado ao orquestrador, delegando decisões de recuperação ao sistema. Esta inversão de controle remove operadores humanos do caminho crítico de recuperação de falhas comuns, permitindo resposta automatizada em escala de segundos ao invés de minutos ou horas típicas de processos manuais.

### 1.2. Conceito de Self-Healing

Self-healing refere-se à capacidade de sistemas detectarem automaticamente problemas e executarem ações corretivas sem intervenção humana. No contexto do Kubernetes, self-healing manifesta-se através de múltiplos mecanismos complementares:

- **Reinicialização automática de containers**: Quando Liveness Probes detectam estados irrecuperáveis
- **Reposicionamento de Pods**: Quando nós falham ou tornam-se indisponíveis
- **Gestão de tráfego**: Removendo automaticamente Pods não-prontos de load balancers
- **Substituição de réplicas**: ReplicaSets e Deployments garantindo número desejado de instâncias saudáveis

Probes constituem mecanismo primário através do qual Kubernetes determina saúde de aplicações, formando entrada fundamental para decisões de self-healing. A eficácia do self-healing depende diretamente da qualidade e adequação das configurações de probes.

### 1.3. Tipos de Probes

Kubernetes oferece três tipos distintos de probes, cada um servindo propósito específico no ciclo de vida de containers:

#### 1.3.1. Startup Probe

Startup Probes verificam se a aplicação dentro de um container iniciou com sucesso, sendo especialmente úteis para containers com tempos de inicialização lentos. Quando configurado, desabilita verificações de liveness e readiness até que seja bem-sucedido, prevenindo que estas probes terminem prematuramente containers ainda no processo de bootstrap.

#### 1.3.2. Readiness Probe

Readiness Probes determinam quando um container está pronto para aceitar tráfego, sendo úteis quando aplicações precisam realizar tarefas iniciais demoradas que dependem de serviços backend, como estabelecer conexões de rede, carregar arquivos ou aquecer caches. Podem também ser úteis posteriormente no ciclo de vida do container, por exemplo, ao recuperar de falhas temporárias ou sobrecargas.

#### 1.3.3. Liveness Probe

O kubelet utiliza Liveness Probes para saber quando reiniciar um container. Por exemplo, liveness probes podem detectar um deadlock, onde uma aplicação está executando mas incapaz de progredir. Reiniciar um container neste estado pode ajudar a tornar a aplicação mais disponível apesar de bugs.

### 1.4. Mecanismos de Verificação

Kubernetes suporta três mecanismos para implementar probes, cada um apropriado para diferentes cenários:

#### 1.4.1. HTTP GET

Executa requisição HTTP GET contra endpoint específico. A probe é considerada bem-sucedida se o código de status HTTP está entre 200 e 399.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome
  initialDelaySeconds: 3
  periodSeconds: 3
```

#### 1.4.2. TCP Socket

Tenta abrir socket TCP para porta especificada. A probe é bem-sucedida se consegue estabelecer conexão.

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```

#### 1.4.3. Exec Command

Executa comando dentro do container. A probe é bem-sucedida se o comando retorna código de saída 0.

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### 1.4.4. gRPC

Verifica saúde através do gRPC Health Checking Protocol.

```yaml
livenessProbe:
  grpc:
    port: 2379
    service: my-service
  initialDelaySeconds: 10
```

## 2. Startup Probes

### 2.1. Propósito e Casos de Uso

Startup Probes endereçam cenário específico onde aplicações requerem tempo significativo para inicialização completa. Bancos de dados podem precisar executar verificações de integridade, aplicações Java podem ter warm-up extenso da JVM, ou serviços podem precisar carregar grandes quantidades de dados na memória antes de estarem operacionais.

Sem Startup Probes, administradores enfrentam dilema: configurar Liveness Probes com `initialDelaySeconds` muito alto desperdiça tempo quando containers falham rapidamente, mas valor muito baixo resulta em terminação prematura de containers que apenas precisam de mais tempo para iniciar. Startup Probes resolvem este problema permitindo configuração generosa especificamente para fase de inicialização, enquanto mantém Liveness Probes responsivas após startup.

### 2.2. Comportamento e Mecânica

Quando uma Startup Probe é configurada, ela desabilita verificações de liveness e readiness até que seja bem-sucedida, garantindo que essas probes não interfiram com a inicialização da aplicação. Este comportamento é crítico para containers com inicialização lenta, evitando que sejam terminados antes de estarem totalmente operacionais.

A Startup Probe é executada apenas uma vez durante a vida do container. Após a primeira verificação bem-sucedida, Liveness e Readiness Probes assumem responsabilidade pelo monitoramento contínuo.

### 2.3. Configuração de Startup Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-startup-pod
spec:
  containers:
  - name: application
    image: myapp:latest
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
      # Permite até 300 segundos (30 * 10) para startup
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

Neste exemplo, a aplicação terá máximo de 5 minutos (30 * 10 = 300s) para finalizar sua inicialização. Uma vez que a startup probe tenha sucesso, a liveness probe assume para fornecer resposta rápida a deadlocks de containers.

### 2.4. Parâmetros de Configuração

failureThreshold determina após quantas falhas consecutivas a probe considera que a verificação falhou. Para startup ou liveness probes, se pelo menos failureThreshold probes falharem, Kubernetes trata o container como não-saudável e aciona reinicialização.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0        # Padrão: 0
  periodSeconds: 10             # Padrão: 10
  timeoutSeconds: 1             # Padrão: 1
  successThreshold: 1           # Deve ser 1 para startup
  failureThreshold: 30          # Padrão: 3
```

### 2.5. Exemplo Prático com Node.js/TypeScript

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthService } from './health.service';

@Controller('health')
export class HealthController {
  private startupComplete: boolean = false;

  constructor(private readonly healthService: HealthService) {
    // Simular inicialização lenta
    this.performStartup();
  }

  private async performStartup() {
    // Simular carregamento de dados, conexões, etc.
    await this.delay(20000); // 20 segundos de startup
    this.startupComplete = true;
  }

  @Get('startup')
  async checkStartup() {
    if (!this.startupComplete) {
      throw new Error('Application still starting up');
    }
    return { status: 'started' };
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 2.6. Troubleshooting de Startup Probes

```bash
# Verificar eventos relacionados a startup probe
kubectl describe pod slow-startup-pod

# Ver logs do container
kubectl logs slow-startup-pod

# Verificar status da startup probe
kubectl get pod slow-startup-pod -o jsonpath='{.status.containerStatuses[0].state}'

# Monitorar eventos em tempo real
kubectl get events --watch --field-selector involvedObject.name=slow-startup-pod
```

Falhas comuns incluem:
- `failureThreshold` muito baixo para tempo real de inicialização
- Endpoint de startup retornando erro durante bootstrap normal
- `timeoutSeconds` insuficiente para latência de resposta da aplicação

## 3. Readiness Probes

### 3.1. Propósito e Filosofia

Readiness Probes informam quando um container está pronto para começar a aceitar tráfego. Um uso comum deste sinal é controlar quais Pods são usados como backends para Services. Quando um Pod não está pronto, é removido dos load balancers do Service.

A distinção crítica entre Readiness e Liveness é que falha de Readiness Probe não causa reinicialização do container - apenas remove o Pod dos endpoints do Service temporariamente até que volte a passar nas verificações. Este comportamento é apropriado para condições temporárias como sobrecarga, aquecimento de caches, ou indisponibilidade transitória de dependências.

### 3.2. Cenários de Aplicação

Readiness Probes são essenciais em múltiplos cenários:

#### 3.2.1. Inicialização Complexa

Aplicações que carregam grandes quantidades de dados, executam migrações de schema, ou estabelecem conexões com múltiplos serviços backend necessitam período após startup antes de estarem prontas para tráfego de produção.

#### 3.2.2. Dependências Externas

Aplicações que dependem criticamente de serviços externos (bancos de dados, caches, APIs) podem usar Readiness Probes para verificar conectividade e disponibilidade destas dependências antes de aceitar tráfego.

#### 3.2.3. Sobrecarga e Recuperação

Durante períodos de alta carga ou ao recuperar de condições adversas, aplicações podem temporariamente sinalizar como não-prontas para prevenir aceitação de tráfego adicional enquanto processam backlog existente.

### 3.3. Configuração de Readiness Probe

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:v2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

### 3.4. Implementação de Endpoints de Readiness

```typescript
// src/health/health.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { CacheService } from '../cache/cache.service';

@Injectable()
export class HealthService {
  constructor(
    private readonly database: DatabaseService,
    private readonly cache: CacheService,
  ) {}

  async checkReadiness(): Promise<boolean> {
    try {
      // Verificar conexão com banco de dados
      await this.database.ping();
      
      // Verificar conexão com cache
      await this.cache.ping();
      
      // Verificar outras dependências críticas
      const externalApiHealthy = await this.checkExternalApi();
      
      return externalApiHealthy;
    } catch (error) {
      console.error('Readiness check failed:', error);
      return false;
    }
  }

  private async checkExternalApi(): Promise<boolean> {
    try {
      const response = await fetch('https://api.external.com/health', {
        timeout: 2000,
      });
      return response.ok;
    } catch {
      return false;
    }
  }
}

// src/health/health.controller.ts
@Controller('health')
export class HealthController {
  constructor(private readonly healthService: HealthService) {}

  @Get('ready')
  async readiness() {
    const isReady = await this.healthService.checkReadiness();
    
    if (!isReady) {
      throw new ServiceUnavailableException('Service not ready');
    }
    
    return {
      status: 'ready',
      timestamp: new Date().toISOString(),
    };
  }
}
```

### 3.5. Readiness vs Liveness

Se quiser enviar tráfego a um Pod apenas quando uma probe for bem-sucedida, especifique uma readiness probe. Neste caso, a readiness probe pode ser a mesma que a liveness probe, mas a existência da readiness probe no spec significa que o Pod iniciará sem receber tráfego e só começará a receber tráfego após a probe começar a ter sucesso.

```yaml
# Exemplo usando probes similares mas com propósitos diferentes
spec:
  containers:
  - name: application
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

O mesmo endpoint pode ser usado, mas com configurações diferentes refletindo diferentes tolerâncias e requisitos.

### 3.6. Biblioteca Terminus para NestJS

A biblioteca @nestjs/terminus facilita implementação de health checks robustos em aplicações NestJS.

```typescript
// Instalação
// npm install @nestjs/terminus

// src/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}

// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HttpHealthIndicator,
  TypeOrmHealthIndicator,
  HealthCheck,
} from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get('ready')
  @HealthCheck()
  checkReadiness() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.http.pingCheck('external-api', 'https://api.external.com/health'),
    ]);
  }

  @Get('live')
  @HealthCheck()
  checkLiveness() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

## 4. Liveness Probes

### 4.1. Propósito e Criticidade

Liveness Probes respondem à questão fundamental: "Este container está vivo e funcional?" O kubelet utiliza liveness probes para saber quando reiniciar um container. Liveness probes podem detectar deadlocks onde uma aplicação está executando mas incapaz de progredir. Reiniciar um container neste estado pode ajudar a tornar a aplicação mais disponível apesar de bugs.

A diferença crucial entre Liveness e Readiness é que falha em Liveness Probe resulta em reinicialização do container, enquanto falha em Readiness Probe apenas remove o Pod dos endpoints do Service temporariamente.

### 4.2. Quando Usar Liveness Probes

Liveness probes devem ser usadas com cautela. Configurações incorretas podem resultar em cascatas de reinicializações que degradam disponibilidade ao invés de melhorá-la. Use liveness probes quando:

- **Aplicação pode entrar em estado irrecuperável**: Deadlocks, corrupção de memória, ou estados onde a aplicação não consegue se recuperar sozinha
- **Aplicação não trava apropriadamente**: Bugs que impedem terminação normal do processo
- **Recuperação através de reinicialização é benéfica**: O overhead de reinicialização é menor que o impacto de manter container defeituoso executando

Não use liveness probes quando:

- **Falhas são temporárias e recuperáveis**: Use readiness probes ao invés
- **Startup é muito lento**: Configure startup probe primeiro
- **Aplicação trava apropriadamente em erros**: O kubelet já detectará e reiniciará

### 4.3. Configuração de Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: application:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

### 4.4. Parâmetros Críticos

#### 4.4.1. initialDelaySeconds

Número de segundos após o container iniciar antes que probes sejam iniciadas. Este parâmetro é crítico para prevenir falsos positivos durante inicialização.

```yaml
# Configuração conservadora para aplicação com startup lento
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 60  # Aguardar 60 segundos antes da primeira verificação
  periodSeconds: 10
```

#### 4.4.2. periodSeconds

Frequência (em segundos) de execução da probe. Padrão é 10 segundos, mínimo é 1.

```yaml
# Verificação mais frequente para detecção rápida
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5  # Verificar a cada 5 segundos
```

#### 4.4.3. failureThreshold

Após probe falhar failureThreshold vezes consecutivas, Kubernetes considera que a verificação global falhou. Para liveness probes, isso aciona reinicialização do container.

```yaml
# Mais tolerante a falhas transitórias
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 5  # Tolerar 5 falhas consecutivas (50 segundos)
```

### 4.5. Implementação de Endpoints de Liveness

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  private isHealthy: boolean = true;
  private lastRequestTime: number = Date.now();

  @Get('healthz')
  async checkLiveness() {
    // Verificar se aplicação está processando requisições
    const now = Date.now();
    const timeSinceLastRequest = now - this.lastRequestTime;
    
    // Se não processou requisições em 5 minutos, pode estar em deadlock
    if (timeSinceLastRequest > 300000) {
      throw new Error('Application appears to be in deadlock');
    }

    // Verificar uso de memória (exemplo simplificado)
    const memoryUsage = process.memoryUsage();
    const heapUsedPercent = (memoryUsage.heapUsed / memoryUsage.heapTotal) * 100;
    
    // Se uso de heap está consistentemente acima de 95%, pode indicar memory leak
    if (heapUsedPercent > 95) {
      throw new Error('Critical memory usage detected');
    }

    return {
      status: 'healthy',
      uptime: process.uptime(),
      memory: memoryUsage,
    };
  }

  // Método chamado por outras partes da aplicação
  updateLastRequestTime() {
    this.lastRequestTime = Date.now();
  }
}
```

### 4.6. Evitando Cascatas de Reinicializações

Configurações inadequadas de liveness probes podem criar situações onde containers são reiniciados continuamente, degradando disponibilidade. Estratégias para evitar:

#### 4.6.1. Configuração Apropriada de Timeouts

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5      # Tempo suficiente para resposta
  failureThreshold: 3    # Tolerar algumas falhas
```

#### 4.6.2. Separação de Concerns

Liveness probes devem verificar apenas saúde da aplicação em si, não dependências externas. Dependências devem ser verificadas por readiness probes.

```typescript
// INCORRETO - Liveness verificando dependências externas
@Get('healthz')
async incorrectLiveness() {
  await this.database.ping();  // NÃO faça isso em liveness!
  await this.cache.ping();
  return { status: 'healthy' };
}

// CORRETO - Liveness verificando apenas estado interno
@Get('healthz')
async correctLiveness() {
  // Verificar se aplicação pode processar requisições
  const canProcess = this.checkInternalState();
  if (!canProcess) {
    throw new Error('Application in bad state');
  }
  return { status: 'healthy' };
}

// CORRETO - Dependências verificadas em readiness
@Get('ready')
async readiness() {
  await this.database.ping();
  await this.cache.ping();
  return { status: 'ready' };
}
```

### 4.7. Monitoramento de Probes

```bash
# Ver eventos de liveness probe
kubectl get events --field-selector reason=Unhealthy

# Descrever pod para ver histórico de probes
kubectl describe pod myapp-pod

# Ver logs do container antes de reinicialização
kubectl logs myapp-pod --previous

# Monitorar reinicializações
kubectl get pod myapp-pod -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

## 5. Uso Combinado de Probes

### 5.1. Estratégia Completa de Health Checking

A combinação apropriada dos três tipos de probes fornece cobertura abrangente para diferentes aspectos do ciclo de vida e saúde de containers.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      containers:
      - name: application
        image: myapp:v3.0
        ports:
        - containerPort: 8080
        
        # Startup Probe - Proteger durante inicialização lenta
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
          # Permite até 300 segundos para startup
        
        # Readiness Probe - Controlar quando receber tráfego
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Liveness Probe - Detectar estados irrecuperáveis
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### 5.2. Implementação de Endpoints Distintos

```typescript
// src/health/health.controller.ts
import { Controller, Get, ServiceUnavailableException } from '@nestjs/common';
import { HealthService } from './health.service';

@Controller('health')
export class HealthController {
  private startupComplete: boolean = false;
  
  constructor(private readonly healthService: HealthService) {
    this.performStartup();
  }

  private async performStartup() {
    try {
      // Simular tarefas de inicialização
      await this.healthService.loadConfiguration();
      await this.healthService.warmupCaches();
      await this.healthService.establishConnections();
      
      this.startupComplete = true;
      console.log('Application startup completed successfully');
    } catch (error) {
      console.error('Startup failed:', error);
      process.exit(1);
    }
  }

  @Get('startup')
  async checkStartup() {
    if (!this.startupComplete) {
      throw new ServiceUnavailableException('Application still starting');
    }
    return {
      status: 'started',
      timestamp: new Date().toISOString(),
    };
  }

  @Get('ready')
  async checkReadiness() {
    // Verificar se aplicação está pronta para processar tráfego
    const checks = await Promise.allSettled([
      this.healthService.checkDatabase(),
      this.healthService.checkCache(),
      this.healthService.checkExternalDependencies(),
    ]);

    const allHealthy = checks.every(
      result => result.status === 'fulfilled' && result.value === true
    );

    if (!allHealthy) {
      throw new ServiceUnavailableException('Service not ready');
    }

    return {
      status: 'ready',
      checks: checks.map(r => r.status),
      timestamp: new Date().toISOString(),
    };
  }

  @Get('live')
  async checkLiveness() {
    // Verificar apenas estado interno da aplicação
    const internalHealth = await this.healthService.checkInternalHealth();
    
    if (!internalHealth.healthy) {
      throw new ServiceUnavailableException(
        `Application unhealthy: ${internalHealth.reason}`
      );
    }

    return {
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
    };
  }
}

// src/health/health.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class HealthService {
  private internalState = {
    healthy: true,
    lastError: null as Error | null,
    consecutiveErrors: 0,
  };

  async loadConfiguration(): Promise<void> {
    // Carregar configurações
    await this.delay(5000);
  }

  async warmupCaches(): Promise<void> {
    // Pré-carregar caches
    await this.delay(3000);
  }

  async establishConnections(): Promise<void> {
    // Estabelecer conexões com serviços externos
    await this.delay(2000);
  }

  async checkDatabase(): Promise<boolean> {
    try {
      // Verificar conexão com banco
      return true;
    } catch (error) {
      console.error('Database check failed:', error);
      return false;
    }
  }

  async checkCache(): Promise<boolean> {
    try {
      // Verificar conexão com cache
      return true;
    } catch (error) {
      console.error('Cache check failed:', error);
      return false;
    }
  }

  async checkExternalDependencies(): Promise<boolean> {
    try {
      // Verificar dependências externas
      return true;
    } catch (error) {
      console.error('External dependency check failed:', error);
      return false;
    }
  }

  async checkInternalHealth(): Promise<{ healthy: boolean; reason?: string }> {
    // Verificar se aplicação está em estado recuperável
    if (this.internalState.consecutiveErrors > 10) {
      return {
        healthy: false,
        reason: 'Too many consecutive errors',
      };
    }

    // Verificar uso de memória
    const memoryUsage = process.memoryUsage();
    const heapUsedPercent = (memoryUsage.heapUsed / memoryUsage.heapTotal) * 100;
    
    if (heapUsedPercent > 98) {
      return {
        healthy: false,
        reason: 'Critical memory usage',
      };
    }

    return { healthy: true };
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 5.3. Padrões de Configuração por Tipo de Aplicação

#### 5.3.1. Aplicações Web Stateless

```yaml
# Aplicação web típica com startup rápido
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 10
  periodSeconds: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

#### 5.3.2. Aplicações com Startup Lento

```yaml
# Aplicação Java com JVM warmup extenso
startupProbe:
  httpGet:
    path: /actuator/health/startup
    port: 8080
  failureThreshold: 60
  periodSeconds: 5
  # Permite até 300 segundos

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 15
  failureThreshold: 3
```

#### 5.3.3. Workers e Processadores em Background

```yaml
# Worker que processa filas
# Pode não ter endpoints HTTP
startupProbe:
  exec:
    command:
    - cat
    - /tmp/started
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:
  exec:
    command:
    - sh
    - -c
    - "queue_length=$(redis-cli llen work_queue); [ $queue_length -lt 1000 ]"
  periodSeconds: 30

livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  periodSeconds: 60
```

## 6. Melhores Práticas e Padrões

### 6.1. Design de Endpoints de Health Check

#### 6.1.1. Endpoints Leves e Rápidos

Health checks devem ser operações baratas que retornam rapidamente. Evite operações pesadas como consultas complexas ao banco de dados.

```typescript
// BOM - Rápido e leve
@Get('healthz')
async checkHealth() {
  return { status: 'ok', timestamp: Date.now() };
}

// RUIM - Operação pesada
@Get('healthz')
async checkHealthBad() {
  const users = await this.db.query('SELECT COUNT(*) FROM users');
  const orders = await this.db.query('SELECT COUNT(*) FROM orders');
  return { users, orders };
}
```

#### 6.1.2. Separação de Responsabilidades

Cada tipo de probe deve verificar aspectos diferentes:

```typescript
// Startup - Verifica se inicialização completou
@Get('startup')
async startup() {
  if (!this.configLoaded || !this.connectionsEstablished) {
    throw new Error('Not started');
  }
  return { status: 'started' };
}

// Readiness - Verifica dependências externas
@Get('ready')
async ready() {
  const dbOk = await this.db.ping();
  const cacheOk = await this.cache.ping();
  
  if (!dbOk || !cacheOk) {
    throw new Error('Dependencies not ready');
  }
  return { status: 'ready' };
}

// Liveness - Verifica apenas estado interno
@Get('live')
async live() {
  const memOk = this.checkMemory();
  const stateOk = this.checkInternalState();
  
  if (!memOk || !stateOk) {
    throw new Error('Internal problems');
  }
  return { status: 'alive' };
}
```

### 6.2. Configuração de Timeouts e Thresholds

#### 6.2.1. Cálculo de Tempo Máximo para Detecção

```
Tempo de detecção = initialDelaySeconds + (failureThreshold * periodSeconds)
```

Exemplo:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3
# Tempo de detecção = 15 + (3 * 10) = 45 segundos
```

#### 6.2.2. Balanceamento de Responsividade vs Estabilidade

```yaml
# Mais responsivo - Detecta problemas rapidamente mas mais sensível
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
  # Detecta em: 10 + (2 * 5) = 20 segundos

# Mais estável - Tolera mais falhas transitórias
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 5
  # Detecta em: 30 + (5 * 10) = 80 segundos
```

### 6.3. Usando Scripts para Verificações Complexas

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - |
      # Script complexo de verificação
      if [ ! -f /tmp/app.pid ]; then
        exit 1
      fi
      
      PID=$(cat /tmp/app.pid)
      if ! kill -0 $PID 2>/dev/null; then
        exit 1
      fi
      
      # Verificar se processo está respondendo
      if ! curl -f http://localhost:8080/health >/dev/null 2>&1; then
        exit 1
      fi
      
      exit 0
  initialDelaySeconds: 30
  periodSeconds: 10
```

### 6.4. Monitoramento e Observabilidade

```typescript
// Adicionar métricas para health checks
import { Injectable } from '@nestjs/common';
import { Counter, Histogram } from 'prom-client';

@Injectable()
export class HealthService {
  private healthCheckCounter = new Counter({
    name: 'health_check_total',
    help: 'Total number of health checks',
    labelNames: ['type', 'status'],
  });

  private healthCheckDuration = new Histogram({
    name: 'health_check_duration_seconds',
    help: 'Duration of health checks',
    labelNames: ['type'],
  });

  async checkHealth(type: 'startup' | 'ready' | 'live'): Promise<boolean> {
    const end = this.healthCheckDuration.startTimer({ type });
    
    try {
      const result = await this.performCheck(type);
      this.healthCheckCounter.inc({ type, status: result ? 'success' : 'failure' });
      return result;
    } finally {
      end();
    }
  }

  private async performCheck(type: string): Promise<boolean> {
    // Implementação da verificação
    return true;
  }
}
```

## 7. Resiliência no Nível de Aplicação

### 7.1. Circuit Breaker Pattern

Circuit Breakers previnem que aplicações façam chamadas repetidas a serviços que estão falhando, permitindo recuperação mais rápida e evitando sobrecarga de sistemas já em dificuldade.

```typescript
// Implementação básica de Circuit Breaker
class CircuitBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failureCount: number = 0;
  private successCount: number = 0;
  private lastFailureTime: number = 0;
  
  constructor(
    private threshold: number = 5,
    private timeout: number = 60000,
    private halfOpenAttempts: number = 3,
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'half-open';
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'half-open') {
      this.successCount++;
      if (this.successCount >= this.halfOpenAttempts) {
        this.state = 'closed';
      }
    }
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.threshold) {
      this.state = 'open';
    }
  }

  getState() {
    return this.state;
  }
}

// Uso em serviço
@Injectable()
export class ExternalApiService {
  private circuitBreaker = new CircuitBreaker(5, 60000, 3);

  async fetchData(): Promise<any> {
    return this.circuitBreaker.call(async () => {
      const response = await fetch('https://external-api.com/data');
      if (!response.ok) {
        throw new Error('API request failed');
      }
      return response.json();
    });
  }

  // Expor estado do circuit breaker para health checks
  isHealthy(): boolean {
    return this.circuitBreaker.getState() !== 'open';
  }
}
```

### 7.2. Fault Injection para Testes

Fault injection permite testar resiliência de aplicações injetando falhas controladas.

```typescript
// Middleware de fault injection
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class FaultInjectionMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Configurável via variáveis de ambiente
    const errorRate = parseFloat(process.env.FAULT_INJECTION_ERROR_RATE || '0');
    const latencyMs = parseInt(process.env.FAULT_INJECTION_LATENCY_MS || '0');

    // Injetar latência
    if (latencyMs > 0) {
      setTimeout(() => this.processRequest(req, res, next, errorRate), latencyMs);
    } else {
      this.processRequest(req, res, next, errorRate);
    }
  }

  private processRequest(
    req: Request,
    res: Response,
    next: NextFunction,
    errorRate: number,
  ) {
    // Injetar erros aleatórios
    if (Math.random() < errorRate) {
      res.status(500).json({
        error: 'Injected fault for testing',
        timestamp: new Date().toISOString(),
      });
      return;
    }

    next();
  }
}
```

### 7.3. Graceful Degradation

Aplicações devem degradar graciosamente quando dependências falham, continuando a fornecer funcionalidade reduzida ao invés de falhar completamente.

```typescript
@Injectable()
export class UserService {
  constructor(
    private database: DatabaseService,
    private cache: CacheService,
    private externalApi: ExternalApiService,
  ) {}

  async getUser(id: string): Promise<User> {
    try {
      // Tentativa 1: Buscar de cache
      const cachedUser = await this.cache.get(`user:${id}`);
      if (cachedUser) {
        return cachedUser;
      }
    } catch (error) {
      console.warn('Cache unavailable, falling back to database');
    }

    try {
      // Tentativa 2: Buscar de banco de dados
      const user = await this.database.findUser(id);
      
      // Tentar enriquecer com dados externos (não-crítico)
      try {
        const externalData = await this.externalApi.getUserData(id);
        user.externalData = externalData;
      } catch (error) {
        console.warn('External API unavailable, returning partial data');
        // Continuar sem dados externos
      }

      return user;
    } catch (error) {
      throw new Error('Unable to retrieve user data');
    }
  }
}
```

## 8. Conclusões

Os mecanismos de probes e self-healing no Kubernetes constituem fundação essencial para operação de aplicações resilientes em ambientes de produção. A capacidade de detectar automaticamente problemas de saúde e executar ações corretivas sem intervenção humana transforma o Kubernetes de simples orquestrador em plataforma verdadeiramente autônoma que melhora dramaticamente disponibilidade e confiabilidade de serviços.

A implementação efetiva requer compreensão profunda das diferenças entre Startup, Readiness e Liveness Probes, cada uma servindo propósito distinto no ciclo de vida de containers. Startup Probes protegem containers com inicialização lenta de terminação prematura, Readiness Probes controlam fluxo de tráfego baseado em prontidão da aplicação e disponibilidade de dependências, e Liveness Probes detectam e remediam estados irrecuperáveis através de reinicialização forçada. O uso apropriado e combinado destes três mecanismos fornece cobertura abrangente para praticamente todos os cenários operacionais.

Configurações inadequadas de probes podem ser prejudiciais, causando cascatas de reinicializações, falsos positivos que degradam disponibilidade, ou falsos negativos que permitem containers defeituosos continuem servindo tráfego. A calibração apropriada de parâmetros como initialDelaySeconds, periodSeconds, timeoutSeconds e failureThreshold requer compreensão das características específicas de cada aplicação, incluindo tempos de inicialização, padrões de falha, e tolerâncias para latência de detecção versus overhead de verificações.

A resiliência verdadeira requer abordagem em camadas que combina self-healing do Kubernetes com padrões de design resiliente no nível de aplicação. Circuit Breakers previnem amplificação de falhas, graceful degradation permite funcionalidade reduzida quando dependências falham, e fault injection possibilita teste sistemático de comportamento sob condições adversas. Esta combinação de capacidades da plataforma com design consciente de aplicação constrói sistemas que não apenas detectam e recuperam de falhas, mas que são projetados desde o início para operar efetivamente mesmo em presença de problemas parciais, oferecendo experiências consistentes aos usuários finais independentemente de turbulências infraestruturais subjacentes.

## 9. Referências Bibliográficas

### 9.1. Documentação Oficial do Kubernetes

- Kubernetes Documentation. "Configure Liveness, Readiness and Startup Probes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

- Kubernetes Documentation. "Pod Lifecycle". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/

- Kubernetes Documentation. "Container probes". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes

### 9.2. Padrões de Resiliência

- NestJS Documentation. "Health Checks (Terminus)". NestJS.com. Disponível em: https://docs.nestjs.com/recipes/terminus

- Cloud Native Computing Foundation. "Cloud Native Patterns". CNCF.io. Disponível em: https://www.cncf.io/

- Microsoft Azure. "Health Endpoint Monitoring Pattern". Microsoft.com. Disponível em: https://docs.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring

### 9.3. Circuit Breakers e Fault Tolerance

- Martin Fowler. "Circuit Breaker". MartinFowler.com. Disponível em: https://martinfowler.com/bliki/CircuitBreaker.html

- Netflix Tech Blog. "Fault Tolerance in a High Volume, Distributed System". NetflixTechBlog.com. Disponível em: https://netflixtechblog.com/

### 9.4. Observabilidade e Monitoramento

- Prometheus Documentation. "Instrumenting HTTP Server". Prometheus.io. Disponível em: https://prometheus.io/docs/guides/go-application/

- Cloud Native Computing Foundation. "OpenTelemetry". OpenTelemetry.io. Disponível em: https://opentelemetry.io/

## 10. Apêndices

### Apêndice A: Exemplos Completos de Configuração

```yaml
# exemplo-completo-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-application
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resilient-app
  template:
    metadata:
      labels:
        app: resilient-app
        version: v1.0
    spec:
      containers:
      - name: application
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: NODE_ENV
          value: production
        
        # Startup Probe - Permite 5 minutos para inicialização
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
            httpHeaders:
            - name: X-Probe-Type
              value: startup
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 30
        
        # Readiness Probe - Verifica dependências
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
            httpHeaders:
            - name: X-Probe-Type
              value: readiness
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        # Liveness Probe - Detecta deadlocks
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
            httpHeaders:
            - name: X-Probe-Type
              value: liveness
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15
      
      terminationGracePeriodSeconds: 30
```

### Apêndice B: Implementação Completa em TypeScript

```typescript
// src/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';
import { HealthService } from './health.service';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
  providers: [HealthService],
  exports: [HealthService],
})
export class HealthModule {}

// src/health/health.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { HealthIndicatorResult } from '@nestjs/terminus';

interface HealthStatus {
  healthy: boolean;
  reason?: string;
  details?: any;
}

@Injectable()
export class HealthService {
  private readonly logger = new Logger(HealthService.name);
  private startupComplete: boolean = false;
  private applicationState = {
    lastError: null as Error | null,
    consecutiveErrors: 0,
    lastSuccessTime: Date.now(),
  };

  constructor() {
    this.initializeApplication();
  }

  private async initializeApplication() {
    try {
      this.logger.log('Starting application initialization...');
      
      // Simular tarefas de inicialização
      await this.loadConfiguration();
      await this.establishConnections();
      await this.warmupCaches();
      await this.runMigrations();
      
      this.startupComplete = true;
      this.logger.log('Application initialization completed successfully');
    } catch (error) {
      this.logger.error('Application initialization failed', error);
      process.exit(1);
    }
  }

  private async loadConfiguration(): Promise<void> {
    this.logger.log('Loading configuration...');
    await this.delay(2000);
  }

  private async establishConnections(): Promise<void> {
    this.logger.log('Establishing database connections...');
    await this.delay(3000);
  }

  private async warmupCaches(): Promise<void> {
    this.logger.log('Warming up caches...');
    await this.delay(5000);
  }

  private async runMigrations(): Promise<void> {
    this.logger.log('Running database migrations...');
    await this.delay(4000);
  }

  async checkStartup(): Promise<HealthStatus> {
    if (!this.startupComplete) {
      return {
        healthy: false,
        reason: 'Application still initializing',
      };
    }

    return {
      healthy: true,
      details: {
        startupTime: new Date().toISOString(),
        uptime: process.uptime(),
      },
    };
  }

  async checkReadiness(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkCache(),
      this.checkMessageQueue(),
      this.checkExternalApi(),
    ]);

    const failures = checks.filter(r => r.status === 'rejected' || !r.value);
    
    if (failures.length > 0) {
      return {
        healthy: false,
        reason: 'One or more dependencies are unavailable',
        details: {
          total: checks.length,
          failures: failures.length,
        },
      };
    }

    return {
      healthy: true,
      details: {
        allDependenciesHealthy: true,
        timestamp: new Date().toISOString(),
      },
    };
  }

  async checkLiveness(): Promise<HealthStatus> {
    // Verificar uso de memória
    const memoryCheck = this.checkMemoryUsage();
    if (!memoryCheck.healthy) {
      return memoryCheck;
    }

    // Verificar se aplicação está travada
    const responsiveCheck = this.checkApplicationResponsiveness();
    if (!responsiveCheck.healthy) {
      return responsiveCheck;
    }

    // Verificar erros consecutivos
    if (this.applicationState.consecutiveErrors > 10) {
      return {
        healthy: false,
        reason: 'Too many consecutive errors',
        details: {
          consecutiveErrors: this.applicationState.consecutiveErrors,
          lastError: this.applicationState.lastError?.message,
        },
      };
    }

    return {
      healthy: true,
      details: {
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        timestamp: new Date().toISOString(),
      },
    };
  }

  private checkMemoryUsage(): HealthStatus {
    const memoryUsage = process.memoryUsage();
    const heapUsedPercent = (memoryUsage.heapUsed / memoryUsage.heapTotal) * 100;

    if (heapUsedPercent > 98) {
      return {
        healthy: false,
        reason: 'Critical memory usage detected',
        details: {
          heapUsedPercent: heapUsedPercent.toFixed(2),
          heapUsed: memoryUsage.heapUsed,
          heapTotal: memoryUsage.heapTotal,
        },
      };
    }

    return { healthy: true };
  }

  private checkApplicationResponsiveness(): HealthStatus {
    const now = Date.now();
    const timeSinceLastSuccess = now - this.applicationState.lastSuccessTime;

    // Se não houve sucesso em 5 minutos, pode estar travada
    if (timeSinceLastSuccess > 300000) {
      return {
        healthy: false,
        reason: 'Application appears unresponsive',
        details: {
          timeSinceLastSuccess: timeSinceLastSuccess,
        },
      };
    }

    return { healthy: true };
  }

  private async checkDatabase(): Promise<boolean> {
    try {
      // Implementar verificação real
      await this.delay(50);
      return true;
    } catch (error) {
      this.logger.error('Database health check failed', error);
      return false;
    }
  }

  private async checkCache(): Promise<boolean> {
    try {
      // Implementar verificação real
      await this.delay(30);
      return true;
    } catch (error) {
      this.logger.error('Cache health check failed', error);
      return false;
    }
  }

  private async checkMessageQueue(): Promise<boolean> {
    try {
      // Implementar verificação real
      await this.delay(40);
      return true;
    } catch (error) {
      this.logger.error('Message queue health check failed', error);
      return false;
    }
  }

  private async checkExternalApi(): Promise<boolean> {
    try {
      // Implementar verificação real
      await this.delay(100);
      return true;
    } catch (error) {
      this.logger.error('External API health check failed', error);
      return false;
    }
  }

  recordSuccess() {
    this.applicationState.lastSuccessTime = Date.now();
    this.applicationState.consecutiveErrors = 0;
  }

  recordError(error: Error) {
    this.applicationState.lastError = error;
    this.applicationState.consecutiveErrors++;
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// src/health/health.controller.ts
import { 
  Controller, 
  Get, 
  ServiceUnavailableException,
  HttpStatus,
} from '@nestjs/common';
import { HealthService } from './health.service';

@Controller('health')
export class HealthController {
  constructor(private readonly healthService: HealthService) {}

  @Get('startup')
  async checkStartup() {
    const result = await this.healthService.checkStartup();
    
    if (!result.healthy) {
      throw new ServiceUnavailableException({
        status: 'unhealthy',
        reason: result.reason,
      });
    }

    return {
      status: 'healthy',
      type: 'startup',
      details: result.details,
    };
  }

  @Get('ready')
  async checkReadiness() {
    const result = await this.healthService.checkReadiness();
    
    if (!result.healthy) {
      throw new ServiceUnavailableException({
        status: 'not_ready',
        reason: result.reason,
        details: result.details,
      });
    }

    return {
      status: 'ready',
      type: 'readiness',
      details: result.details,
    };
  }

  @Get('live')
  async checkLiveness() {
    const result = await this.healthService.checkLiveness();
    
    if (!result.healthy) {
      throw new ServiceUnavailableException({
        status: 'unhealthy',
        reason: result.reason,
        details: result.details,
      });
    }

    return {
      status: 'healthy',
      type: 'liveness',
      details: result.details,
    };
  }
}
```

### Apêndice C: Scripts de Verificação Bash

```bash
#!/bin/bash
# health-check.sh - Script complexo de health check

set -e

# Configurações
HEALTH_FILE="/tmp/healthy"
PID_FILE="/tmp/app.pid"
LOG_FILE="/var/log/app.log"
MAX_ERROR_RATE=0.1
MAX_MEMORY_PERCENT=90

# Função para verificar se processo está rodando
check_process() {
    if [ ! -f "$PID_FILE" ]; then
        echo "PID file not found"
        return 1
    fi
    
    PID=$(cat "$PID_FILE")
    if ! kill -0 "$PID" 2>/dev/null; then
        echo "Process not running"
        return 1
    fi
    
    return 0
}

# Função para verificar endpoint HTTP
check_http_endpoint() {
    local url="$1"
    local timeout="${2:-5}"
    
    if ! curl -f -s --max-time "$timeout" "$url" > /dev/null; then
        echo "HTTP endpoint check failed: $url"
        return 1
    fi
    
    return 0
}

# Função para verificar uso de memória
check_memory() {
    local used_percent
    used_percent=$(free | grep Mem | awk '{print ($3/$2) * 100.0}')
    
    if (( $(echo "$used_percent > $MAX_MEMORY_PERCENT" | bc -l) )); then
        echo "Memory usage too high: ${used_percent}%"
        return 1
    fi
    
    return 0
}

# Função para verificar taxa de erro em logs
check_error_rate() {
    if [ ! -f "$LOG_FILE" ]; then
        return 0
    fi
    
    local total_lines
    local error_lines
    local error_rate
    
    total_lines=$(tail -n 1000 "$LOG_FILE" | wc -l)
    error_lines=$(tail -n 1000 "$LOG_FILE" | grep -c "ERROR" || true)
    
    if [ "$total_lines" -eq 0 ]; then
        return 0
    fi
    
    error_rate=$(echo "scale=2; $error_lines / $total_lines" | bc)
    
    if (( $(echo "$error_rate > $MAX_ERROR_RATE" | bc -l) )); then
        echo "Error rate too high: ${error_rate}"
        return 1
    fi
    
    return 0
}

# Função principal
main() {
    local check_type="${1:-liveness}"
    
    case "$check_type" in
        startup)
            if [ ! -f "$HEALTH_FILE" ]; then
                echo "Application not started yet"
                exit 1
            fi
            ;;
        
        readiness)
            check_process || exit 1
            check_http_endpoint "http://localhost:8080/health/ready" || exit 1
            ;;
        
        liveness)
            check_process || exit 1
            check_http_endpoint "http://localhost:8080/health/live" 2 || exit 1
            check_memory || exit 1
            check_error_rate || exit 1
            ;;
        
        *)
            echo "Unknown check type: $check_type"
            exit 1
            ;;
    esac
    
    echo "Health check passed: $check_type"
    exit 0
}

main "$@"
```

### Apêndice D: Comandos de Troubleshooting

```bash
# Verificar status de probes
kubectl get pod <pod-name> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
kubectl get pod <pod-name> -o jsonpath='{.status.conditions[?(@.type=="ContainersReady")].status}'

# Ver eventos relacionados a probes
kubectl get events --field-selector involvedObject.name=<pod-name>
kubectl describe pod <pod-name> | grep -A 10 "Events:"

# Verificar contagem de reinicializações
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].restartCount}'

# Ver logs do container atual
kubectl logs <pod-name>

# Ver logs do container anterior (antes de reinicialização)
kubectl logs <pod-name> --previous

# Monitorar eventos de probes em tempo real
kubectl get events --watch | grep -i probe

# Testar endpoint de health check manualmente
kubectl exec <pod-name> -- curl -f http://localhost:8080/health/live

# Ver configuração de probes
kubectl get pod <pod-name> -o yaml | grep -A 20 "livenessProbe:"
kubectl get pod <pod-name> -o yaml | grep -A 20 "readinessProbe:"
kubectl get pod <pod-name> -o yaml | grep -A 20 "startupProbe:"

# Verificar histórico de mudanças de estado
kubectl get pod <pod-name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'

# Estatísticas de saúde de todos os Pods
kubectl get pods -o custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,RESTARTS:.status.containerStatuses[0].restartCount
```

### Apêndice E: Glossário e Termos Técnicos

**Circuit Breaker**: Padrão de design que previne aplicações de fazer chamadas repetidas a serviços que estão falhando, permitindo recuperação mais rápida.

**Deadlock**: Situação onde processo ou thread está bloqueado indefinidamente aguardando recurso que nunca será liberado.

**Degradation (Graceful Degradation)**: Estratégia onde aplicação continua fornecendo funcionalidade reduzida quando dependências ou recursos críticos falham.

**Failure Threshold**: Número de falhas consecutivas de probe necessárias antes que Kubernetes considere verificação como falha.

**Fault Injection**: Técnica de teste onde falhas são injetadas intencionalmente em sistema para validar comportamento de recuperação.

**Health Check**: Verificação periódica executada para determinar se aplicação ou serviço está saudável e operacional.

**initialDelaySeconds**: Número de segundos após container iniciar antes que primeira probe seja executada.

**Liveness Probe**: Verificação que determina se container está vivo e funcional, acionando reinicialização se falhar.

**Mean Time To Recovery (MTTR)**: Tempo médio necessário para recuperar de falha, importante métrica de resiliência.

**periodSeconds**: Frequência (em segundos) com que probe é executada após a primeira verificação.

**Readiness Probe**: Verificação que determina se container está pronto para receber tráfego, controlando inclusão em endpoints de Service.

**Self-Healing**: Capacidade de sistema detectar e corrigir automaticamente problemas sem intervenção humana.

**Startup Probe**: Verificação executada durante inicialização de container para determinar se aplicação iniciou com sucesso.

**successThreshold**: Número mínimo de sucessos consecutivos para probe ser considerada bem-sucedida após falha.

**timeoutSeconds**: Número de segundos após o qual probe é considerada falha se não retornar resposta.