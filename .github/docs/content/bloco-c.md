<!-- markdownlint-disable -->

# Bloco C - Deployment e Configuração de Aplicações em Produção

## Resumo Executivo

A transição de conceitos teóricos do Kubernetes para implementação prática de aplicações reais requer compreensão profunda não apenas dos recursos fundamentais da plataforma, mas também das melhores práticas de desenvolvimento, containerização, gestão de configurações e estratégias de deployment que garantam operações confiáveis e sustentáveis em ambientes de produção. Este documento explora sistematicamente o ciclo completo de implantação de aplicações no Kubernetes, desde a containerização até gestão de configurações sensíveis, utilizando aplicação NestJS como referência para demonstrar padrões e práticas aplicáveis a qualquer stack tecnológica.

O processo de containerização constitui ponto de partida fundamental, transformando aplicações em artefatos portáveis e imutáveis através de Docker images que encapsulam código, dependências e configurações de runtime. A construção de images otimizadas, minimizando tamanho e maximizando eficiência de caching de layers, impacta diretamente velocidade de deployments, utilização de rede e armazenamento em registries. A gestão apropriada de tags de imagens emerge como prática crítica: enquanto tags mutáveis como `latest` oferecem conveniência durante desenvolvimento, introduzem riscos substanciais em produção através de indeterminismo e impossibilidade de rollbacks precisos, motivando uso de tags imutáveis baseadas em commit hashes ou semantic versioning.

Estratégias de deployment no Kubernetes determinam como novas versões de aplicações substituem versões antigas, equilibrando disponibilidade, velocidade de rollout e consumo de recursos. Rolling updates, estratégia padrão do Kubernetes, substituem Pods gradualmente através de parâmetros configuráveis `maxSurge` e `maxUnavailable` que controlam quantos Pods adicionais podem ser criados temporariamente e quantos podem estar indisponíveis durante atualização. A estratégia Recreate, que termina todos os Pods antigos antes de criar novos, embora cause downtime completo, pode ser necessária para aplicações que não suportam múltiplas versões executando simultaneamente ou que requerem migrações de schema incompatíveis.

A separação de configuração de código através de ConfigMaps para dados não-sensíveis e Secrets para informações confidenciais representa princípio fundamental de aplicações cloud-native, permitindo mesma imagem de container executar em múltiplos ambientes com configurações distintas. Esta abordagem facilita promoção de aplicações através de pipelines de CI/CD, simplifica gestão de configurações específicas de ambiente, e melhora segurança através de controle granular de acesso a informações sensíveis. A injeção de configurações pode ocorrer através de variáveis de ambiente individuais, importação completa de ConfigMaps/Secrets, ou montagem como volumes, cada abordagem apropriada para diferentes cenários e requisitos de aplicação.

## 1. Introdução e Conceitos

### 1.1. Do Desenvolvimento Local ao Cluster Kubernetes

A jornada de uma aplicação desde ambiente de desenvolvimento local até execução em cluster Kubernetes de produção envolve múltiplas transformações e considerações que transcendem simplesmente "fazer funcionar no Kubernetes". Aplicações desenvolvidas localmente tipicamente assumem presença de recursos específicos (bases de dados, caches, serviços externos) acessíveis através de configurações hard-coded ou arquivos de ambiente locais. A containerização força explicitação de todas as dependências e configurações, enquanto deployment em Kubernetes adiciona camadas de abstração, resiliência e escalabilidade que requerem design consciente desde concepção da aplicação.

O paradigma de aplicações cloud-native enfatiza características como statelessness, externalização de configuração, observabilidade através de logs estruturados e métricas, e design para falha que assume components individuais podem e irão falhar, requerendo que aplicações implementem graceful degradation e recovery. Esta mudança de mentalidade - de aplicações como entidades monolíticas e persistentes para componentes efêmeros e substituíveis - fundamenta práticas modernas de deployment e operação em Kubernetes.

### 1.2. Containerização como Fundação

Containers proporcionam unidade padrão de empacotamento que encapsula aplicação e todas suas dependências, garantindo consistência entre ambientes de desenvolvimento, teste e produção. Diferentemente de virtualização tradicional onde cada máquina virtual inclui sistema operacional completo, containers compartilham kernel do host, resultando em overhead significativamente menor e tempos de inicialização medidos em segundos ao invés de minutos.

Docker tornou-se runtime de containers de facto, embora Kubernetes suporte outras implementações através da Container Runtime Interface (CRI). A criação de container images através de Dockerfiles define processo reproduzível que transforma código-fonte em artefato deployável, documentando simultaneamente dependências e processo de build da aplicação.

```dockerfile
# Exemplo de Dockerfile multi-stage para aplicação Node.js
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/main"]
```

### 1.3. Registries de Container: Armazenamento e Distribuição

Container registries servem como repositórios centralizados para armazenar, gerenciar e distribuir container images. Docker Hub constitui registry público mais conhecido, mas organizações frequentemente utilizam registries privados (AWS ECR, Google Container Registry, Azure Container Registry, Harbor) para manter controle sobre images proprietárias e implementar políticas de segurança e compliance.

O fluxo típico envolve build de image localmente ou em pipeline de CI, push para registry, e pull posterior pelo kubelet em cada nó quando Pods são schedulados. Este modelo distribuído permite que qualquer nó no cluster obtenha images necessárias sem dependência de armazenamento centralizado compartilhado.

```bash
# Autenticação com Docker Hub
docker login

# Build de image com tag
docker build -t username/application:v1.0.0 .

# Push para registry
docker push username/application:v1.0.0

# Pull em outro ambiente (executado automaticamente pelo kubelet)
docker pull username/application:v1.0.0
```

## 2. Containerização de Aplicações

### 2.1. Estrutura de Projeto NestJS

NestJS é framework Node.js para construir aplicações server-side eficientes e escaláveis, utilizando TypeScript por padrão e incorporando conceitos de arquiteturas modulares e injeção de dependências. A estrutura típica de projeto NestJS organiza código em modules, controllers e services.

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Configuração de porta via variável de ambiente
  const port = process.env.PORT || 3000;
  
  await app.listen(port);
  console.log(`Application is running on: http://localhost:${port}`);
}

bootstrap();
```

```typescript
// src/app.controller.ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get('health')
  getHealth(): object {
    return {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    };
  }
}
```

### 2.2. Dockerfile Otimizado

A construção de Dockerfiles otimizados requer compreensão de como Docker processa layers e utiliza cache para acelerar builds subsequentes. Cada instrução em Dockerfile cria nova layer, e Docker reutiliza layers cacheadas quando instruções e contexto não mudaram.

```dockerfile
# Dockerfile multi-stage para aplicação NestJS

# Stage 1: Build
FROM node:18-alpine AS builder

# Instalar dependências necessárias para build
RUN apk add --no-cache python3 make g++

# Definir diretório de trabalho
WORKDIR /app

# Copiar arquivos de dependências primeiro (melhor caching)
COPY package.json package-lock.json ./

# Instalar dependências de produção e desenvolvimento
RUN npm ci

# Copiar código fonte
COPY . .

# Build da aplicação
RUN npm run build

# Remover dev dependencies
RUN npm prune --production

# Stage 2: Runtime
FROM node:18-alpine

# Criar usuário não-root para segurança
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

# Definir diretório de trabalho
WORKDIR /app

# Copiar node_modules e build do stage anterior
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/package*.json ./

# Mudar para usuário não-root
USER nestjs

# Expor porta
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1); })"

# Comando de inicialização
CMD ["node", "dist/main.js"]
```

### 2.3. Dockerignore para Build Eficiente

O arquivo `.dockerignore` previne que arquivos desnecessários sejam incluídos no contexto de build, reduzindo tamanho e tempo de build.

```
# .dockerignore
node_modules
npm-debug.log
dist
.git
.gitignore
README.md
.env
.env.*
*.md
.vscode
.idea
coverage
.nyc_output
```

### 2.4. Build e Push de Images

```bash
# Build de image
docker build -t myusername/nestjs-app:v1.0.0 .

# Verificar image criada
docker images | grep nestjs-app

# Tag adicional (latest deve ser evitado em produção)
docker tag myusername/nestjs-app:v1.0.0 myusername/nestjs-app:latest

# Login no Docker Hub
docker login

# Push para registry
docker push myusername/nestjs-app:v1.0.0
docker push myusername/nestjs-app:latest

# Testar image localmente
docker run -p 3000:3000 myusername/nestjs-app:v1.0.0
```

### 2.5. Otimizações de Tamanho de Image

```dockerfile
# Usar base image alpine (significativamente menor)
FROM node:18-alpine

# Multi-stage builds para separar build de runtime
FROM node:18 AS builder
# ... build steps ...
FROM node:18-alpine
COPY --from=builder /app/dist ./dist

# Remover arquivos desnecessários
RUN npm prune --production && \
    rm -rf /root/.npm /tmp/*

# Usar .dockerignore apropriadamente
```

Comparação de tamanhos:
- `node:18` (Debian-based): ~900MB
- `node:18-alpine`: ~150MB
- Multi-stage optimized: ~100-150MB

## 3. Deployment no Kubernetes

### 3.1. Manifesto de Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
  namespace: production
  labels:
    app: nestjs-app
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nestjs-app
  template:
    metadata:
      labels:
        app: nestjs-app
        version: v1.0.0
    spec:
      containers:
      - name: nestjs-app
        image: myusername/nestjs-app:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### 3.2. Service para Exposição

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nestjs-app-service
  namespace: production
  labels:
    app: nestjs-app
spec:
  type: ClusterIP
  selector:
    app: nestjs-app
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
```

### 3.3. Aplicação e Verificação

```bash
# Criar namespace
kubectl create namespace production

# Aplicar manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Verificar deployment
kubectl get deployment -n production
kubectl get pods -n production
kubectl get service -n production

# Descrever recursos
kubectl describe deployment nestjs-app -n production
kubectl describe pod <pod-name> -n production

# Ver logs
kubectl logs -n production <pod-name>
kubectl logs -n production <pod-name> --follow

# Port forward para teste
kubectl port-forward -n production service/nestjs-app-service 8080:80

# Testar aplicação
curl http://localhost:8080
curl http://localhost:8080/health
```

## 4. Gestão de Tags de Imagens

### 4.1. O Problema com Tags Mutáveis

Tags como `latest` são mutáveis - o mesmo nome de tag pode apontar para diferentes images ao longo do tempo. Esta mutabilidade introduz múltiplos problemas:

- **Indeterminismo**: Impossível saber qual versão específica de image está executando
- **Rollbacks problemáticos**: `kubectl rollout undo` pode não reverter para versão esperada
- **Inconsistência entre ambientes**: Diferentes nós podem ter versões diferentes cacheadas
- **Auditoria impossível**: Não há registro claro de qual código está em produção

```yaml
# EVITAR em produção
spec:
  containers:
  - name: app
    image: myapp:latest  # Problema: qual versão exatamente?
    imagePullPolicy: Always  # Força pull mesmo que tag seja mesmo
```

### 4.2. Image Pull Policy

O `imagePullPolicy` determina quando kubelet deve pullar image do registry:

- **Always**: Sempre pull image, mesmo se já existe localmente
- **IfNotPresent**: Pull apenas se image não existe localmente
- **Never**: Nunca pull, use apenas image local (falhará se não existir)

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    imagePullPolicy: IfNotPresent  # Recomendado com tags específicas
```

Comportamento padrão:
- Tag `:latest` ou sem tag: `imagePullPolicy: Always`
- Tag específica: `imagePullPolicy: IfNotPresent`

### 4.3. Tags Imutáveis Baseadas em Semantic Versioning

```bash
# Formato: <registry>/<repository>:<major>.<minor>.<patch>
docker build -t myusername/nestjs-app:1.0.0 .
docker push myusername/nestjs-app:1.0.0
```

```yaml
spec:
  containers:
  - name: app
    image: myusername/nestjs-app:1.0.0  # Tag imutável
    imagePullPolicy: IfNotPresent
```

### 4.4. Tags Baseadas em Git Commit

```bash
# Obter commit hash
COMMIT_HASH=$(git rev-parse --short HEAD)

# Build com commit hash
docker build -t myusername/nestjs-app:${COMMIT_HASH} .
docker push myusername/nestjs-app:${COMMIT_HASH}

# Também criar tag de versão
docker tag myusername/nestjs-app:${COMMIT_HASH} myusername/nestjs-app:1.0.0
docker push myusername/nestjs-app:1.0.0
```

```yaml
spec:
  containers:
  - name: app
    image: myusername/nestjs-app:a3f8c2d  # Commit hash garante rastreabilidade exata
    imagePullPolicy: IfNotPresent
```

### 4.5. Atualização de Image em Deployment

```bash
# Método 1: Kubectl set image
kubectl set image deployment/nestjs-app \
  nestjs-app=myusername/nestjs-app:v1.1.0 \
  -n production

# Método 2: Edit inline
kubectl edit deployment nestjs-app -n production
# Alterar spec.template.spec.containers[0].image

# Método 3: Patch
kubectl patch deployment nestjs-app -n production \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nestjs-app","image":"myusername/nestjs-app:v1.1.0"}]}}}}'

# Método 4 (RECOMENDADO): Atualizar manifesto e aplicar
# Editar deployment.yaml alterando image tag
kubectl apply -f deployment.yaml
```

## 5. Estratégias de Deployment

### 5.1. Rolling Update

Rolling Update é estratégia padrão que substitui Pods antigos por novos gradualmente, mantendo disponibilidade durante processo.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3        # Máximo de 3 Pods adicionais durante update
      maxUnavailable: 2  # Máximo de 2 Pods indisponíveis durante update
  template:
    # ... spec do Pod
```

#### 5.1.1. Parâmetros de Rolling Update

**maxSurge**: Número máximo de Pods que podem ser criados além de `replicas` durante update. Pode ser número absoluto ou percentual.

```yaml
maxSurge: 3      # 3 Pods adicionais
maxSurge: 25%    # 25% de répl
```

Com `replicas: 10` e `maxSurge: 3`, pode haver até 13 Pods durante update.

**maxUnavailable**: Número máximo de Pods que podem estar indisponíveis durante update.

```yaml
maxUnavailable: 2    # 2 Pods podem estar down
maxUnavailable: 10%  # 10% das réplicas podem estar down
```

Com `replicas: 10` e `maxUnavailable: 2`, pelo menos 8 Pods devem estar disponíveis.

#### 5.1.2. Configurações para Zero Downtime

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # Criar 1 novo antes de terminar antigo
    maxUnavailable: 0     # Zero downtime garantido
```

#### 5.1.3. Configurações para Update Rápido

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 50%         # Criar muitos Pods novos rapidamente
    maxUnavailable: 25%   # Pode ter mais indisponibilidade
```

### 5.2. Recreate Strategy

Estratégia Recreate termina todos os Pods antigos antes de criar novos, causando downtime completo mas garantindo que apenas uma versão execute por vez.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-migration-app
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    # ... spec do Pod
```

#### 5.2.1. Casos de Uso para Recreate

- Aplicações que não suportam múltiplas versões simultaneamente
- Migrações de schema de banco de dados incompatíveis
- Aplicações stateful que requerem shutdown limpo
- Desenvolvimento e staging onde downtime é aceitável

### 5.3. Estratégias Avançadas (Não Nativas)

#### 5.3.1. Blue-Green Deployment

Mantém dois ambientes idênticos (Blue = produção atual, Green = nova versão). Traffic é switchado instantaneamente de Blue para Green após validação.

```yaml
# Deployment Blue (versão antiga)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0

---
# Deployment Green (nova versão)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2.0.0

---
# Service inicialmente apontando para Blue
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue  # Switch para 'green' para mudar versão
  ports:
  - port: 80
    targetPort: 8080
```

#### 5.3.2. Canary Deployment

Libera nova versão gradualmente para subconjunto de usuários, permitindo validação em produção com impacto limitado.

```yaml
# Deployment principal (versão estável) - 90% do tráfego
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1.0.0

---
# Deployment canary (nova versão) - 10% do tráfego
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2.0.0

---
# Service balanceia entre stable e canary
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp  # Seleciona ambos stable e canary
  ports:
  - port: 80
    targetPort: 8080
```

### 5.4. Rollback de Deployments

```bash
# Ver histórico de rollouts
kubectl rollout history deployment/nestjs-app -n production

# Ver detalhes de revisão específica
kubectl rollout history deployment/nestjs-app -n production --revision=2

# Fazer rollback para revisão anterior
kubectl rollout undo deployment/nestjs-app -n production

# Fazer rollback para revisão específica
kubectl rollout undo deployment/nestjs-app -n production --to-revision=3

# Verificar status do rollout
kubectl rollout status deployment/nestjs-app -n production

# Pausar rollout (para fazer múltiplas mudanças)
kubectl rollout pause deployment/nestjs-app -n production

# Retomar rollout pausado
kubectl rollout resume deployment/nestjs-app -n production

# Restart deployment (recria todos os Pods)
kubectl rollout restart deployment/nestjs-app -n production
```

### 5.5. Controle de Histórico de Revisões

```yaml
spec:
  revisionHistoryLimit: 10  # Manter 10 ReplicaSets antigos para rollback
```

Por padrão, Kubernetes mantém 10 ReplicaSets antigos. Reduzir este número economiza recursos mas limita histórico de rollback disponível.

## 6. ConfigMaps: Gestão de Configurações

### 6.1. Conceito e Propósito

ConfigMaps permitem desacoplar configuração de images de containers, tornando aplicações mais portáveis e facilitando gestão de configurações específicas de ambiente. Mesma image pode executar em desenvolvimento, staging e produção com diferentes ConfigMaps.

### 6.2. Criação de ConfigMaps

#### 6.2.1. A Partir de Literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_NAME="NestJS Application" \
  --from-literal=APP_ENV="production" \
  --from-literal=LOG_LEVEL="info" \
  -n production
```

#### 6.2.2. A Partir de Arquivo

```bash
# Criar arquivo de configuração
cat > app.properties << EOF
APP_NAME=NestJS Application
APP_ENV=production
LOG_LEVEL=info
DATABASE_HOST=postgres.production.svc.cluster.local
DATABASE_PORT=5432
EOF

# Criar ConfigMap a partir de arquivo
kubectl create configmap app-config \
  --from-file=app.properties \
  -n production
```

#### 6.2.3. A Partir de Diretório

```bash
# Criar ConfigMap de todos os arquivos em diretório
kubectl create configmap app-config \
  --from-file=./config/ \
  -n production
```

#### 6.2.4. Declarativamente via YAML

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: nestjs-app
data:
  APP_NAME: "NestJS Application"
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  # Configurações multi-linha
  application.yaml: |
    server:
      port: 3000
      host: 0.0.0.0
    logging:
      level: info
      format: json
```

```bash
kubectl apply -f configmap.yaml
```

### 6.3. Consumindo ConfigMaps em Pods

#### 6.3.1. Como Variáveis de Ambiente Individuais

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    env:
    - name: APP_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_NAME
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
```

#### 6.3.2. Importando Todas as Keys como Variáveis

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    envFrom:
    - configMapRef:
        name: app-config
```

Todas as chaves do ConfigMap tornam-se variáveis de ambiente no container.

#### 6.3.3. Como Volume Montado

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Cada chave no ConfigMap torna-se arquivo em `/etc/config/`.

#### 6.3.4. Montando Keys Específicas

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: application.yaml
        path: app.yaml
```

### 6.4. Atualização de ConfigMaps

```bash
# Editar ConfigMap existente
kubectl edit configmap app-config -n production

# Ou atualizar via arquivo
kubectl apply -f configmap.yaml

# Verificar mudanças
kubectl describe configmap app-config -n production
```

**Importante**: Mudanças em ConfigMaps não acionam reinicialização automática de Pods. Para aplicar mudanças:

```bash
# Método 1: Restart deployment
kubectl rollout restart deployment/nestjs-app -n production

# Método 2: Forçar update adicionando annotation
kubectl patch deployment nestjs-app -n production \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"configmap-update\":\"$(date)\"}}}}}"
```

## 7. Secrets: Gestão de Informações Sensíveis

### 7.1. Conceito e Diferenças de ConfigMaps

Secrets são objetos para armazenar informações sensíveis como senhas, tokens OAuth, chaves SSH. Embora similares a ConfigMaps em uso, Secrets têm tratamento especial:

- Valores são base64-encoded (não criptografados por padrão)
- Podem ser criptografados at-rest se configurado
- Acesso pode ser restrito via RBAC mais granular
- Kubernetes evita escrever Secrets em disco quando possível

### 7.2. Tipos de Secrets

- **Opaque** (padrão): Dados arbitrários definidos pelo usuário
- **kubernetes.io/service-account-token**: Token para ServiceAccount
- **kubernetes.io/dockercfg**: Credenciais Docker registry (formato antigo)
- **kubernetes.io/dockerconfigjson**: Credenciais Docker registry (formato novo)
- **kubernetes.io/basic-auth**: Credenciais para autenticação básica
- **kubernetes.io/ssh-auth**: Dados para autenticação SSH
- **kubernetes.io/tls**: Certificado TLS e chave privada

### 7.3. Criação de Secrets

#### 7.3.1. A Partir de Literals

```bash
kubectl create secret generic database-credentials \
  --from-literal=username=admin \
  --from-literal=password='SuperSecret123!' \
  -n production
```

#### 7.3.2. A Partir de Arquivos

```bash
# Criar arquivo com senha
echo -n 'SuperSecret123!' > password.txt

# Criar Secret
kubectl create secret generic database-credentials \
  --from-file=password=./password.txt \
  -n production

# Limpar arquivo local
rm password.txt
```

#### 7.3.3. Declarativamente via YAML

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
data:
  # Valores devem ser base64-encoded
  username: YWRtaW4=  # 'admin' em base64
  password: U3VwZXJTZWNyZXQxMjMh  # 'SuperSecret123!' em base64
```

```bash
# Encodar valores
echo -n 'admin' | base64
echo -n 'SuperSecret123!' | base64

# Aplicar Secret
kubectl apply -f secret.yaml
```

#### 7.3.4. Usando stringData (Mais Legível)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
stringData:
  # stringData aceita valores em texto plano
  # Kubernetes converte automaticamente para base64
  username: admin
  password: SuperSecret123!
```

### 7.4. Consumindo Secrets em Pods

#### 7.4.1. Como Variáveis de Ambiente

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    env:
    - name: DATABASE_USERNAME
      valueFrom:
        secretKeyRef:
          name: database-credentials
          key: username
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: database-credentials
          key: password
```

#### 7.4.2. Importando Todas as Keys

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    envFrom:
    - secretRef:
        name: database-credentials
```

#### 7.4.3. Como Volume Montado

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: database-credentials
      defaultMode: 0400  # Permissões read-only para owner
```

### 7.5. Exemplo Completo com Aplicação NestJS

```yaml
# deployment-with-secrets.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nestjs-app
  template:
    metadata:
      labels:
        app: nestjs-app
    spec:
      containers:
      - name: app
        image: myusername/nestjs-app:v1.0.0
        ports:
        - containerPort: 3000
        env:
        # Variáveis de ConfigMap
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_NAME
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        # Variáveis de Secret
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
```

### 7.6. Gestão Avançada de Secrets

#### 7.6.1. Secrets Externos com Vault

HashiCorp Vault oferece gestão centralizada de secrets com criptografia, rotação automática e auditoria.

```yaml
# Usando Vault Agent Injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-vault
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/database"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/database" -}}
          export DATABASE_USERNAME="{{ .Data.data.username }}"
          export DATABASE_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      # Container spec...
```

#### 7.6.2. Sealed Secrets

Sealed Secrets permite armazenar secrets criptografados em Git.

```bash
# Instalar kubeseal
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Criar secret regular
kubectl create secret generic mysecret \
  --from-literal=password=SuperSecret123! \
  --dry-run=client -o yaml > mysecret.yaml

# Selar secret
kubeseal -f mysecret.yaml -w mysealedsecret.yaml

# mysealedsecret.yaml pode ser commitado no Git
# Controller no cluster descriptografa automaticamente
kubectl apply -f mysealedsecret.yaml
```

### 7.7. Boas Práticas com Secrets

1. **Nunca committar Secrets em Git**: Usar `.gitignore`
2. **Usar RBAC para limitar acesso**: Apenas ServiceAccounts necessárias devem ter acesso
3. **Habilitar encryption at-rest**: Configurar no API Server
4. **Rotacionar secrets regularmente**: Implementar processo de rotação
5. **Usar soluções externas para produção**: Vault, AWS Secrets Manager, etc.
6. **Auditar acesso a secrets**: Logs e monitoring
7. **Limitar escopo**: Secrets por namespace, não cluster-wide

```yaml
# Exemplo de RBAC para limitar acesso a Secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["database-credentials"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-database-credentials
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## 8. Conclusões

A jornada de implantação de aplicações reais no Kubernetes transcende simplesmente "fazer funcionar" para abranger práticas sustentáveis de operação, manutenção e evolução de sistemas em produção. A containerização apropriada através de Dockerfiles otimizados estabelece fundação para deployments eficientes, onde tamanho de images, estratégias de caching de layers e separação de stages de build e runtime impactam diretamente velocidade de deployments e utilização de recursos em clusters.

A gestão disciplinada de tags de images emerge como prática crítica que diferencia ambientes casuais de operações profissionais de produção. Tags imutáveis baseadas em semantic versioning ou commit hashes proporcionam rastreabilidade exata, rollbacks previsíveis e eliminam classes inteiras de problemas relacionados a indeterminismo e inconsistência entre ambientes. O abandono de práticas convenientes mas problemáticas como uso de tag `latest` em produção representa maturidade operacional necessária para sistemas confiáveis.

Estratégias de deployment no Kubernetes oferecem controle granular sobre como novas versões substituem antigas, equilibrando disponibilidade, velocidade e consumo de recursos. Rolling updates com configurações apropriadas de `maxSurge` e `maxUnavailable` permitem zero-downtime deployments para aplicações stateless, enquanto estratégia Recreate, apesar de causar indisponibilidade temporária, pode ser necessária para aplicações com requisitos específicos de consistência ou migrações de estado. Estratégias avançadas como blue-green e canary deployments, embora não nativas do Kubernetes, podem ser implementadas através de manipulação cuidadosa de labels, selectors e Services, oferecendo capacidades adicionais para validação progressiva de releases em produção.

A externalização de configuração através de ConfigMaps e Secrets representa princípio fundamental de aplicações cloud-native que facilita portabilidade, simplifica gestão de configurações específicas de ambiente, e melhora segurança através de separação de dados sensíveis de código. A capacidade de consumir configurações através de variáveis de ambiente, montagem de volumes, ou importação completa de ConfigMaps/Secrets oferece flexibilidade para diferentes padrões de aplicação e requisitos operacionais. Para ambientes de produção, integração com soluções especializadas de gestão de secrets como HashiCorp Vault ou AWS Secrets Manager proporciona capacidades adicionais de criptografia, rotação automática, auditoria e controle de acesso que excedem capacidades nativas de Secrets do Kubernetes, estabelecendo fundações para operações seguras e compliant em escala empresarial.

## 9. Referências Bibliográficas

### 9.1. Documentação Oficial do Kubernetes

- Kubernetes Documentation. "Deployments". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

- Kubernetes Documentation. "ConfigMaps". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/configuration/configmap/

- Kubernetes Documentation. "Secrets". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/configuration/secret/

- Kubernetes Documentation. "Images". Kubernetes.io. Disponível em: https://kubernetes.io/docs/concepts/containers/images/

### 9.2. Docker e Containerização

- Docker Documentation. "Dockerfile Best Practices". Docker.com. Disponível em: https://docs.docker.com/develop/dev-best-practices/

- Docker Documentation. "Multi-stage Builds". Docker.com. Disponível em: https://docs.docker.com/develop/develop-images/multistage-build/

### 9.3. NestJS

- NestJS Documentation. "NestJS Official Documentation". NestJS.com. Disponível em: https://docs.nestjs.com/

### 9.4. Gestão de Secrets

- HashiCorp Vault Documentation. "Vault on Kubernetes". Vaultproject.io. Disponível em: https://www.vaultproject.io/docs/platform/k8s

- Bitnami Sealed Secrets. "Sealed Secrets". GitHub. Disponível em: https://github.com/bitnami-labs/sealed-secrets

### 9.5. Estratégias de Deployment

- Kubernetes Blog. "Deployment Strategies". Kubernetes.io/blog. Disponível em: https://kubernetes.io/blog/

- Martin Fowler. "Blue-Green Deployment". MartinFowler.com. Disponível em: https://martinfowler.com/bliki/BlueGreenDeployment.html

## 10. Apêndices

### Apêndice A: Exemplo Completo de Aplicação NestJS

```typescript
// src/config/configuration.ts
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  app: {
    name: process.env.APP_NAME || 'NestJS App',
    environment: process.env.APP_ENV || 'development',
  },
  database: {
    host: process.env.DATABASE_HOST || 'localhost',
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432,
    username: process.env.DATABASE_USERNAME,
    password: process.env.DATABASE_PASSWORD,
    database: process.env.DATABASE_NAME || 'appdb',
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '1d',
  },
  external: {
    apiKey: process.env.API_KEY,
    apiUrl: process.env.API_URL,
  },
});

// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  
  const port = configService.get('port');
  const appName = configService.get('app.name');
  
  await app.listen(port);
  console.log(`${appName} is running on: http://localhost:${port}`);
}

bootstrap();
```

### Apêndice B: Manifests Kubernetes Completos

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
    environment: production

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_NAME: "NestJS Production App"
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "appdb"
  JWT_EXPIRES_IN: "7d"
  API_URL: "https://api.external.com"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_USERNAME: "appuser"
  DATABASE_PASSWORD: "SuperSecretPassword123!"
  JWT_SECRET: "your-super-secret-jwt-key-change-in-production"
  API_KEY: "your-external-api-key"

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
  namespace: production
  labels:
    app: nestjs-app
    version: v1.0.0
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nestjs-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nestjs-app
        version: v1.0.0
    spec:
      containers:
      - name: nestjs-app
        image: myusername/nestjs-app:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      terminationGracePeriodSeconds: 30

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nestjs-app-service
  namespace: production
  labels:
    app: nestjs-app
spec:
  type: ClusterIP
  selector:
    app: nestjs-app
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
```

### Apêndice C: Scripts de Deploy

```bash
#!/bin/bash
# deploy.sh - Script automatizado de deploy

set -e

# Configurações
IMAGE_NAME="myusername/nestjs-app"
NAMESPACE="production"
VERSION="${1:-v1.0.0}"

echo "=== Deploying $IMAGE_NAME:$VERSION to $NAMESPACE ==="

# Build image
echo "Building Docker image..."
docker build -t $IMAGE_NAME:$VERSION .

# Push to registry
echo "Pushing image to registry..."
docker push $IMAGE_NAME:$VERSION

# Update deployment
echo "Updating Kubernetes deployment..."
kubectl set image deployment/nestjs-app \
  nestjs-app=$IMAGE_NAME:$VERSION \
  -n $NAMESPACE

# Wait for rollout
echo "Waiting for rollout to complete..."
kubectl rollout status deployment/nestjs-app -n $NAMESPACE

# Verify deployment
echo "Verifying deployment..."
kubectl get pods -n $NAMESPACE -l app=nestjs-app

echo "=== Deploy completed successfully ==="
```

### Apêndice D: Glossário e Termos Técnicos

**Base64**: Esquema de encoding que representa dados binários em formato ASCII string, usado para encodar valores em Secrets do Kubernetes.

**Blue-Green Deployment**: Estratégia que mantém dois ambientes idênticos onde tráfego é switchado instantaneamente entre versões.

**Canary Deployment**: Estratégia que libera nova versão gradualmente para subconjunto de usuários antes de rollout completo.

**ConfigMap**: Objeto do Kubernetes para armazenar dados de configuração não-confidenciais em pares chave-valor.

**Container Registry**: Repositório para armazenar e distribuir container images (Docker Hub, ECR, GCR, ACR).

**Dockerfile**: Arquivo de texto contendo instruções para construir container image.

**envFrom**: Campo de especificação de Pod que importa todas as chaves de ConfigMap ou Secret como variáveis de ambiente.

**Image Pull Policy**: Política que determina quando kubelet deve pullar container image do registry (Always, IfNotPresent, Never).

**Immutable Tag**: Tag de container image que não muda ao longo do tempo, garantindo rastreabilidade e rollbacks previsíveis.

**maxSurge**: Parâmetro de rolling update que especifica número máximo de Pods adicionais que podem ser criados durante atualização.

**maxUnavailable**: Parâmetro de rolling update que especifica número máximo de Pods que podem estar indisponíveis durante atualização.

**Multi-stage Build**: Técnica de Dockerfile que usa múltiplos stages FROM para otimizar tamanho final de image.

**Mutable Tag**: Tag de container image que pode apontar para diferentes images ao longo do tempo (exemplo: `latest`).

**NestJS**: Framework Node.js progressivo para construir aplicações server-side eficientes e escaláveis.

**Opaque Secret**: Tipo padrão de Secret para dados arbitrários definidos pelo usuário.

**Recreate**: Estratégia de deployment que termina todos os Pods antigos antes de criar novos, causando downtime.

**Registry**: Serviço para armazenar e distribuir container images, como Docker Hub, ECR, GCR ou ACR.

**Revision History**: Histórico de ReplicaSets antigos mantidos para possibilitar rollback de Deployments.

**Rollback**: Processo de reverter Deployment para versão anterior após problemas detectados.

**Rolling Update**: Estratégia padrão de deployment que substitui Pods antigos por novos gradualmente, mantendo disponibilidade.

**Sealed Secrets**: Solução que permite armazenar secrets criptografados em sistemas de controle de versão.

**Secret**: Objeto do Kubernetes para armazenar informações sensíveis como senhas, tokens e chaves.

**Semantic Versioning**: Sistema de versionamento usando formato MAJOR.MINOR.PATCH (exemplo: v1.2.3).

**stringData**: Campo de Secret que aceita valores em texto plano, convertidos automaticamente para base64.

**Vault**: Solução da HashiCorp para gestão centralizada de secrets com criptografia e rotação automática.

**Zero Downtime Deployment**: Estratégia de atualização que mantém aplicação disponível durante todo processo de deployment.
