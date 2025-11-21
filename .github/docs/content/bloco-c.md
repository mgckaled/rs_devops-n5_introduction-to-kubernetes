<!-- markdownlint-disable -->

# Kubernetes: Deployment e Configuração de Aplicações em Produção

## 1. Resumo Executivo

Este documento apresenta os conceitos e práticas essenciais para deployment de aplicações reais no Kubernetes, abordando desde a containerização até o gerenciamento de configurações sensíveis. O foco está em demonstrar o ciclo completo de uma aplicação em produção, incluindo build de imagens Docker, versionamento adequado, gerenciamento de configurações através de ConfigMaps e Secrets, estratégias avançadas de deployment e práticas de rollback. O conteúdo explora problemas comuns encontrados em ambientes produtivos, como mutabilidade de tags de imagem, gerenciamento de variáveis de ambiente e escolha de estratégias de deployment apropriadas. Através de uma abordagem teórica fundamentada em casos práticos, o documento estabelece as bases para operações confiáveis, seguras e escaláveis de aplicações containerizadas no Kubernetes.

## 2. Introdução e Conceitos

### 2.1. Ciclo de Vida de Aplicações no Kubernetes

O deployment de aplicações no Kubernetes envolve múltiplas etapas que devem ser compreendidas e executadas corretamente para garantir confiabilidade e segurança.

#### Fases do Ciclo

1. **Desenvolvimento**: Criação da aplicação e suas dependências
2. **Containerização**: Empacotamento em imagem Docker
3. **Registro**: Armazenamento em Container Registry
4. **Deployment**: Implantação no cluster Kubernetes
5. **Configuração**: Injeção de variáveis e secrets
6. **Exposição**: Disponibilização via Services
7. **Monitoramento**: Observabilidade e métricas
8. **Atualização**: Rollout de novas versões
9. **Rollback**: Reversão em caso de problemas

### 2.2. Containerização de Aplicações

#### Conceito de Container

Containers encapsulam aplicações e todas as suas dependências em uma unidade executável consistente. No contexto do Kubernetes, a qualidade da imagem Docker impacta diretamente:

- **Tempo de deployment**: Imagens menores são transferidas mais rapidamente
- **Consumo de recursos**: Imagens otimizadas usam menos disco e memória
- **Superfície de ataque**: Menos componentes reduzem vulnerabilidades
- **Tempo de startup**: Imagens enxutas inicializam mais rápido

#### Dockerfile: Anatomia

```dockerfile
# Imagem base
FROM node:18-alpine AS builder

# Diretório de trabalho
WORKDIR /app

# Cópia de arquivos de dependências
COPY package*.json ./

# Instalação de dependências
RUN npm ci --only=production

# Cópia do código fonte
COPY . .

# Build da aplicação
RUN npm run build

# Estágio de produção
FROM node:18-alpine

WORKDIR /app

# Cópia de dependências e build do estágio anterior
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Exposição de porta
EXPOSE 3000

# Usuário não-root (segurança)
USER node

# Comando de execução
CMD ["node", "dist/main.js"]
```

#### Componentes do Dockerfile

**FROM**: Imagem base
- Use versões específicas (não latest)
- Prefira imagens alpine (menores)
- Considere multi-stage builds

**WORKDIR**: Diretório de trabalho
- Define contexto de execução
- Organiza estrutura de arquivos

**COPY**: Cópia de arquivos
- Ordem importa para cache de layers
- Copie dependências antes do código

**RUN**: Execução de comandos
- Minimize número de layers
- Combine comandos relacionados
- Limpe cache após instalações

**EXPOSE**: Documentação de porta
- Não abre porta automaticamente
- Informativo para usuários da imagem

**USER**: Usuário de execução
- Nunca use root em produção
- Crie usuário dedicado ou use node

**CMD/ENTRYPOINT**: Comando de inicialização
- CMD: pode ser sobrescrito
- ENTRYPOINT: ponto de entrada fixo

#### Multi-stage Builds

Técnica para reduzir tamanho final da imagem:

```dockerfile
# Estágio 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Estágio 2: Produção (somente artefatos necessários)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/main.js"]
```

**Benefícios**:
- Imagem final contém apenas artefatos de produção
- Ferramentas de build não incluídas
- Redução drástica de tamanho (GB → MB)

### 2.3. Container Registry

#### Função e Importância

Container Registry é repositório centralizado para armazenar e distribuir imagens Docker.

**Provedores Principais**:
- **Docker Hub**: Público e privado, gratuito com limitações
- **Amazon ECR**: Integrado com AWS
- **Google GCR**: Integrado com GCP
- **Azure ACR**: Integrado com Azure
- **GitHub Container Registry**: Integrado com GitHub
- **Harbor**: Self-hosted, open source

#### Workflow de Registry

```bash
# 1. Build da imagem
docker build -t myapp:1.0.0 .

# 2. Tag com endereço do registry
docker tag myapp:1.0.0 username/myapp:1.0.0

# 3. Login no registry
docker login

# 4. Push da imagem
docker push username/myapp:1.0.0
```

#### Autenticação do Kubernetes

Para pull de imagens privadas, Kubernetes usa ImagePullSecrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

Referência no Deployment:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: registry-credentials
      containers:
      - name: app
        image: private-registry.com/myapp:1.0.0
```

### 2.4. Versionamento de Imagens

#### Estratégias de Versionamento

**Semantic Versioning** (recomendado):

```text
MAJOR.MINOR.PATCH
1.0.0 → 1.0.1 (patch: bug fix)
1.0.1 → 1.1.0 (minor: nova feature)
1.1.0 → 2.0.0 (major: breaking change)
```

**Commit-based**:

```bash
# Tag baseada em commit hash
docker build -t myapp:abc1234 .
docker build -t myapp:commit-abc1234 .
```

**Timestamp**:

```bash
# Tag com data/hora
docker build -t myapp:2024-01-15-143000 .
```

**Branch-based** (ambientes):

```bash
docker build -t myapp:develop .
docker build -t myapp:staging .
docker build -t myapp:production-v1.2.3 .
```

#### Problema da Tag Latest

**Por que evitar latest**:

1. **Não-determinístico**: Latest muda sem aviso
2. **Impossível rastrear**: Qual versão está rodando?
3. **Rollback inviável**: Não há versão anterior definida
4. **Cache problemático**: ImagePullPolicy comporta-se diferente
5. **Ambiguidade**: Latest local ≠ latest remote

**Exemplo de problema**:

```yaml
# Deployment com tag latest
spec:
  containers:
  - name: app
    image: myapp:latest  # PROBLEMA!
    imagePullPolicy: IfNotPresent  # Não puxa se já existe localmente
```

Se node já tem myapp:latest local, nova versão não será puxada mesmo com nova build.

### 2.5. ImagePullPolicy

Controla quando Kubernetes puxa imagem do registry.

#### Políticas Disponíveis

**Always**:

```yaml
imagePullPolicy: Always
```

- Sempre puxa imagem do registry
- Ignora cache local
- Garante versão mais recente
- Mais lento (network overhead)
- **Recomendado**: Tag latest ou mutável

**IfNotPresent** (padrão para tags específicas):

```yaml
imagePullPolicy: IfNotPresent
```

- Puxa apenas se não existe localmente
- Usa cache quando possível
- Mais rápido
- **Problema**: Não detecta mudanças em tag mutável
- **Recomendado**: Tags imutáveis (v1.2.3, commit-abc123)

**Never**:

```yaml
imagePullPolicy: Never
```

- Nunca puxa do registry
- Usa apenas imagens locais
- Útil para desenvolvimento local
- **Nunca use em produção**

#### Comportamento Padrão

Kubernetes define automaticamente baseado na tag:

```yaml
# Tag específica → IfNotPresent
image: myapp:1.2.3
# imagePullPolicy: IfNotPresent (implícito)

# Tag latest → Always
image: myapp:latest
# imagePullPolicy: Always (implícito)

# Sem tag → Always
image: myapp
# imagePullPolicy: Always (implícito)
```

#### Melhores Práticas

1. **Use tags imutáveis** (commit hash, semantic version)
2. **Especifique imagePullPolicy explicitamente**
3. **IfNotPresent com tags específicas** (performance)
4. **Always apenas com latest** (se realmente necessário)
5. **Nunca mude conteúdo de tag existente**

## 3. ConfigMaps: Gerenciamento de Configuração

### 3.1. Conceito e Propósito

ConfigMap é objeto Kubernetes para armazenar dados de configuração não sensíveis em pares chave-valor.

#### Casos de Uso

- **Variáveis de ambiente**: URLs, portas, nomes
- **Arquivos de configuração**: config.json, application.yml
- **Scripts**: Inicialização, healthchecks
- **Parâmetros de aplicação**: Features flags, timeouts
- **Configurações específicas de ambiente**: dev, staging, prod

#### Separação de Responsabilidades

```text
Código (Imagem Docker) → O QUE fazer
ConfigMap → COMO fazer (configuração)
```

**Benefícios**:
- Mesma imagem, diferentes configurações
- Mudança sem rebuild
- Configuração versionável (Git)
- Facilitação de ambientes múltiplos

### 3.2. Criação de ConfigMaps

#### Imperativo: a partir de literais

```bash
kubectl create configmap app-config \
  --from-literal=APP_NAME=MyApplication \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
```

#### Imperativo: a partir de arquivo

Arquivo `app.env`:

```env
APP_NAME=MyApplication
APP_ENV=production
LOG_LEVEL=info
DATABASE_HOST=postgres.default.svc.cluster.local
```

```bash
kubectl create configmap app-config --from-file=app.env
```

#### Imperativo: a partir de diretório

```bash
kubectl create configmap nginx-config --from-file=./config-files/
```

#### Declarativo: YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_NAME: "MyApplication"
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
```

### 3.3. Consumindo ConfigMaps

#### Variável de Ambiente Individual

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
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
```

**Características**:
- Granular: Seleciona chaves específicas
- Renomeável: Nome da env ≠ chave do ConfigMap
- Explícito: Fica claro quais variáveis são usadas

#### Todas as Variáveis (envFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    envFrom:
    - configMapRef:
        name: app-config
```

**Características**:
- Todas as chaves do ConfigMap viram variáveis de ambiente
- Nome da variável = chave do ConfigMap
- Mais simples para muitas variáveis
- Menos controle sobre nomes

#### Prefixo em envFrom

```yaml
envFrom:
- prefix: "CONFIG_"
  configMapRef:
    name: app-config
```

Resultado: `CONFIG_APP_NAME`, `CONFIG_LOG_LEVEL`, etc.

#### ConfigMap como Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

**Resultado**:
```text
/etc/config/APP_NAME → conteúdo: MyApplication
/etc/config/APP_ENV → conteúdo: production
/etc/config/LOG_LEVEL → conteúdo: info
```

**Uso**: Arquivos de configuração complexos (JSON, YAML, XML)

#### Volume com Subpath Específico

```yaml
volumeMounts:
- name: config
  mountPath: /app/config.json
  subPath: config.json
volumes:
- name: config
  configMap:
    name: app-config
    items:
    - key: app-config.json
      path: config.json
```

### 3.4. Atualização de ConfigMaps

#### Atualização de ConfigMap Existente

```bash
kubectl edit configmap app-config
```

Ou aplicar novo manifesto:

```bash
kubectl apply -f configmap.yaml
```

#### Propagação para Pods

**Volume**: Atualização automática (após ~1min)
- kubelet detecta mudança
- Atualiza arquivos montados
- Aplicação deve reload configuração

**Environment Variables**: NÃO atualiza automaticamente
- Variáveis definidas na criação do Pod
- Necessário recriar Pods (rollout restart)

```bash
# Forçar restart de Deployment
kubectl rollout restart deployment/myapp
```

### 3.5. ConfigMap Opcional

```yaml
env:
- name: OPTIONAL_CONFIG
  valueFrom:
    configMapKeyRef:
      name: optional-config
      key: value
      optional: true  # Não falha se ConfigMap não existir
```

### 3.6. Imutabilidade de ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "MyApp"
immutable: true  # Não pode ser modificado
```

**Benefícios**:
- Proteção contra mudanças acidentais
- Performance (kubelet não monitora mudanças)
- Segurança (prevenção de alterações maliciosas)

**Limitação**: Necessário criar novo ConfigMap para mudanças

## 4. Secrets: Gerenciamento de Dados Sensíveis

### 4.1. Conceito e Propósito

Secrets armazenam informações sensíveis (senhas, tokens, chaves) de forma mais segura que ConfigMaps.

#### Diferenças ConfigMap vs Secret

| Aspecto | ConfigMap | Secret |
|---------|-----------|---------|
| Propósito | Configuração não sensível | Dados sensíveis |
| Codificação | Texto plano | Base64 |
| Armazenamento etcd | Plano (default) | Pode ser criptografado |
| RBAC | Mesmo que outros recursos | Mais restritivo recomendado |
| Uso | Configs gerais | Credenciais, tokens, chaves |

#### Tipos de Secrets

**Opaque** (genérico):

```yaml
type: Opaque
```

**kubernetes.io/dockerconfigjson** (pull secrets):

```yaml
type: kubernetes.io/dockerconfigjson
```

**kubernetes.io/tls** (certificados TLS):

```yaml
type: kubernetes.io/tls
```

**kubernetes.io/service-account-token**:

```yaml
type: kubernetes.io/service-account-token
```

### 4.2. Criação de Secrets

#### Imperativo: literais

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123
```

#### Imperativo: arquivo

```bash
# Arquivo db-password.txt contém a senha
kubectl create secret generic db-password \
  --from-file=password=./db-password.txt
```

#### Declarativo: YAML (valores em base64)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=        # "admin" em base64
  password: U3VwZXJTZWNyZXQxMjM=  # "SuperSecret123" em base64
```

**Encoding base64**:

```bash
echo -n 'admin' | base64
# YWRtaW4=

echo -n 'SuperSecret123' | base64
# U3VwZXJTZWNyZXQxMjM=
```

**Decoding**:

```bash
echo 'YWRtaW4=' | base64 -d
# admin
```

#### stringData (texto plano no manifesto)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Kubernetes converte para base64
  username: admin
  password: SuperSecret123
```

**Atenção**: stringData é conveniência, mas expõe segredo em manifesto

### 4.3. Consumindo Secrets

#### Variável de Ambiente Individual

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

#### Todas as Chaves (envFrom)

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    envFrom:
    - secretRef:
        name: db-credentials
```

Resultado: Variáveis `username` e `password`

#### Secret como Volume

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    volumeMounts:
    - name: db-secrets
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: db-secrets
    secret:
      secretName: db-credentials
```

Resultado:
```text
/secrets/username → admin
/secrets/password → SuperSecret123
```

**Vantagem**: Secrets em volumes são automaticamente atualizados

### 4.4. Segurança de Secrets

#### Limitações do Base64

Base64 **NÃO é criptografia**, é apenas encoding:

```bash
kubectl get secret db-credentials -o yaml
# Qualquer pessoa com acesso pode decodificar
```

#### Práticas de Segurança

1. **Encryption at Rest**: Habilitar criptografia do etcd

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
    - identity: {}
```

2. **RBAC Restritivo**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["db-credentials"]  # Apenas este secret
```

3. **Namespaces Separados**: Isolar secrets por ambiente

4. **Secret Managers Externos**:
   - **HashiCorp Vault**
   - **AWS Secrets Manager**
   - **Azure Key Vault**
   - **Google Secret Manager**

#### Vault Integration

```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-db: "secret/data/database/config"
```

Vault injeta secrets dinamicamente sem armazená-los no Kubernetes.

### 4.5. Secrets Imutáveis

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: U3VwZXJTZWNyZXQxMjM=
immutable: true
```

**Benefícios**: Mesmos do ConfigMap imutável

## 5. Combinando env e envFrom

### 5.1. Múltiplas Fontes

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    env:
    # Variável direta
    - name: NODE_ENV
      value: "production"
    # De ConfigMap
    - name: APP_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_NAME
    # De Secret
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-secrets
          key: key
    envFrom:
    # Todas de ConfigMap
    - configMapRef:
        name: app-config
    # Todas de Secret
    - secretRef:
        name: db-credentials
```

### 5.2. Precedência de Variáveis

Quando mesma variável definida em múltiplas fontes:

```yaml
env:
- name: LOG_LEVEL
  value: "debug"  # Define primeiro

envFrom:
- configMapRef:
    name: app-config  # Contém LOG_LEVEL: "info"
```

**Resultado**: `env` tem precedência sobre `envFrom`
- LOG_LEVEL = "debug"

**Ordem de precedência** (maior para menor):
1. `env` (variáveis diretas)
2. `envFrom` (última ocorrência prevalece)

### 5.3. Validação de Variáveis

#### Problema

`envFrom` injeta TODAS as chaves. Se aplicação espera `DATABASE_URL` mas ConfigMap tem `DB_URL`, aplicação falhará sem variável esperada.

#### Solução

1. **Documentar variáveis esperadas**: README, schema
2. **Validação na aplicação**: Startup checks
3. **Testes de integração**: Verificar variáveis necessárias
4. **Preferir env explícito** quando crítico

```typescript
// Validação de variáveis na aplicação
const requiredEnvVars = [
  'DATABASE_URL',
  'API_KEY',
  'APP_NAME'
];

requiredEnvVars.forEach(varName => {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
});
```

## 6. Estratégias Avançadas de Deployment

### 6.1. Rolling Update (Padrão)

Já abordado no Bloco B, reiteramos conceitos com foco em produção.

#### Configuração Otimizada

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3        # Permite 30% a mais temporariamente
      maxUnavailable: 1  # Apenas 10% indisponível por vez
  minReadySeconds: 30    # Aguarda 30s antes de considerar Ready
  progressDeadlineSeconds: 600  # Timeout de 10min
```

**Análise**:
- maxSurge: 3 → Pode ter 13 Pods temporariamente (10 + 3)
- maxUnavailable: 1 → Mínimo de 9 Pods sempre disponíveis
- minReadySeconds: 30 → Detecta problemas de startup
- progressDeadlineSeconds: 600 → Evita rollouts travados

### 6.2. Recreate

Deleta todos Pods antes de criar novos.

#### Configuração

```yaml
spec:
  strategy:
    type: Recreate
```

#### Fluxo de Execução

1. Escala ReplicaSet antigo para 0
2. Aguarda todos Pods terminarem
3. Cria ReplicaSet novo
4. Escala para número de réplicas desejado

#### Quando Usar

**Casos de uso apropriados**:
- Aplicação não suporta múltiplas versões simultâneas
- Banco de dados com migrações incompatíveis
- Recursos compartilhados não suportam lock (filesystem)
- Ambientes de desenvolvimento/teste

**Evitar em produção devido a**:
- Downtime completo
- Experiência de usuário degradada
- Sem capacidade de rollback gradual

### 6.3. Blue-Green Deployment

Duas versões completas rodando simultaneamente, switch instantâneo de tráfego.

#### Arquitetura

```text
Service → Label Selector: version=blue

Deployment Blue (version=blue)  ← Tráfego atual
  └─ 10 Pods v1.0

Deployment Green (version=green) ← Nova versão em standby
  └─ 10 Pods v2.0
```

#### Implementação

**Deployment Blue** (versão atual):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 10
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
      - name: myapp
        image: myapp:1.0.0
```

**Deployment Green** (nova versão):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 10
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
      - name: myapp
        image: myapp:2.0.0
```

**Service** (aponta para Blue):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Tráfego para Blue
  ports:
  - port: 80
    targetPort: 3000
```

#### Processo de Deploy

1. Deploy versão Green (nova)
2. Teste versão Green (smoke tests, health checks)
3. Switch Service para Green: `kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'`
4. Monitorar versão Green em produção
5. Se OK: Deletar Blue
6. Se problema: Rollback instantâneo para Blue

#### Vantagens

- **Zero downtime**: Switch instantâneo
- **Rollback imediato**: Reverter selector
- **Teste em produção**: Green pode ser testado antes de receber tráfego
- **Segurança**: Versão anterior pronta para rollback

#### Desvantagens

- **Custo**: Dobro de recursos temporariamente
- **Complexidade**: Gerenciar dois Deployments
- **Banco de dados**: Migrações podem ser desafiadoras

### 6.4. Canary Deployment

Release gradual para subset de usuários.

#### Arquitetura

```text
Service → Seleciona app=myapp (sem version)

Deployment Stable (version=stable)
  └─ 9 Pods v1.0 (90% tráfego)

Deployment Canary (version=canary)
  └─ 1 Pod v2.0 (10% tráfego)
```

#### Implementação

**Deployment Stable**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
```

**Deployment Canary**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 10% do tráfego (1 de 10 pods)
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
```

**Service** (balanceia entre todos):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Seleciona AMBOS stable e canary
  ports:
  - port: 80
    targetPort: 3000
```

#### Progressão Gradual

1. **10%**: 1 Canary, 9 Stable
2. **Monitorar**: Métricas, erros, latência
3. **25%**: 3 Canary, 7 Stable
4. **50%**: 5 Canary, 5 Stable
5. **75%**: 7 Canary, 3 Stable
6. **100%**: 10 Canary, 0 Stable → Renomear Canary para Stable

#### Ferramentas Avançadas

**Flagger** (Progressive Delivery):

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      threshold: 99
      interval: 1m
```

Flagger automatiza progressão baseada em métricas.

#### Vantagens

- **Redução de risco**: Exposição gradual
- **Detecção precoce**: Problemas afetam poucos usuários
- **Rollback fácil**: Apenas escalar Canary para 0
- **A/B testing**: Comparar versões em produção

#### Desvantagens

- **Complexidade**: Múltiplos Deployments
- **Monitoramento**: Necessário para decisões
- **Duração**: Deploy completo leva mais tempo

### 6.5. Comparação de Estratégias

| Estratégia | Downtime | Complexidade | Custo | Rollback | Caso de Uso |
|------------|----------|--------------|-------|----------|-------------|
| RollingUpdate | Zero | Baixa | Normal | Gradual | Padrão, maioria apps |
| Recreate | Sim | Baixa | Normal | Rebuild | Dev, apps incompatíveis |
| Blue-Green | Zero | Média | 2x | Instantâneo | Lançamentos críticos |
| Canary | Zero | Alta | 1.1-1.5x | Gradual | Alta criticidade, A/B |

## 7. Rollback e Histórico

### 7.1. Histórico de Rollout

Kubernetes mantém histórico de revisões:

```bash
kubectl rollout history deployment/myapp
```

Output:
```text
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Update image to v1.1.0
3         Update image to v1.2.0
```

#### Anotações para Change-Cause

```bash
# Deployment com anotação
kubectl apply -f deployment.yaml --record
```

Ou no manifesto:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Deploy version 1.2.0 with new feature X"
```

### 7.2. Inspeção de Revisão

```bash
# Ver detalhes da revisão 2
kubectl rollout history deployment/myapp --revision=2
```

Output mostra template do Pod naquela revisão.

### 7.3. Rollback

#### Rollback para Revisão Anterior

```bash
kubectl rollout undo deployment/myapp
```

Kubernetes reverte para revisão imediatamente anterior.

#### Rollback para Revisão Específica

```bash
kubectl rollout undo deployment/myapp --to-revision=2
```

Reverte para revisão 2 especificamente.

#### Processo de Rollback

1. Kubernetes identifica ReplicaSet da revisão target
2. Inicia rolling update reverso
3. Escala ReplicaSet antigo (da revisão) para cima
4. Escala ReplicaSet atual para baixo
5. Cria nova revisão do rollback no histórico

### 7.4. Pausar e Retomar Rollout

```bash
# Pausar rollout em andamento
kubectl rollout pause deployment/myapp

# Fazer múltiplas mudanças
kubectl set image deployment/myapp app=myapp:2.0.0
kubectl set resources deployment/myapp -c=app --limits=cpu=500m,memory=512Mi

# Retomar rollout (aplica todas mudanças de uma vez)
kubectl rollout resume deployment/myapp
```

**Uso**: Agrupar múltiplas mudanças em único rollout

### 7.5. Limite de Histórico

```yaml
spec:
  revisionHistoryLimit: 10  # Mantém 10 ReplicaSets antigas
```

**Default**: 10

**Ajustar baseado em**:
- Frequência de deploys
- Necessidade de rollback profundo
- Consumo de disco (etcd)

## 8. Gestão de Configuração em Múltiplos Ambientes

### 8.1. Estratégias de Organização

#### Por Namespace

```bash
# ConfigMaps por ambiente
kubectl apply -f configmap.yaml -n development
kubectl apply -f configmap.yaml -n staging
kubectl apply -f configmap.yaml -n production
```

Mesmo nome, namespaces diferentes, valores diferentes.

#### Por Nome de Recurso

```yaml
# development
name: app-config-dev

# production
name: app-config-prod
```

Deployment referencia apropriado:

```yaml
envFrom:
- configMapRef:
    name: app-config-${ENV}  # Substituído por CI/CD
```

### 8.2. Kustomize

Ferramenta nativa para customização de manifestos Kubernetes.

#### Estrutura

```text
base/
  deployment.yaml
  service.yaml
  kustomization.yaml
overlays/
  development/
    kustomization.yaml
    configmap.yaml
  production/
    kustomization.yaml
    configmap.yaml
    replicas-patch.yaml
```

#### Base

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
```

`base/kustomization.yaml`:

```yaml
resources:
- deployment.yaml
- service.yaml
```

#### Overlay de Produção

`overlays/production/kustomization.yaml`:

```yaml
bases:
- ../../base

configMapGenerator:
- name: app-config
  literals:
  - APP_ENV=production
  - LOG_LEVEL=warn

patchesStrategicMerge:
- replicas-patch.yaml

images:
- name: myapp
  newTag: 1.2.0
```

`overlays/production/replicas-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
```

#### Build e Deploy

```bash
# Gerar manifestos finais
kustomize build overlays/production

# Aplicar diretamente
kubectl apply -k overlays/production
```

### 8.3. Helm

Gerenciador de pacotes Kubernetes.

#### Chart Structure

```text
myapp/
  Chart.yaml
  values.yaml
  values-dev.yaml
  values-prod.yaml
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml
```

#### Template com Variáveis

`templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        env:
        - name: APP_ENV
          value: {{ .Values.environment }}
```

#### Values

`values-prod.yaml`:

```yaml
replicaCount: 10
environment: production
image:
  repository: myapp
  tag: "1.2.0"
```

#### Deploy

```bash
helm install myapp ./myapp -f values-prod.yaml -n production
```

## 9. Conclusões

### Principais Aprendizados

1. **Containerização Eficiente**:
   - Multi-stage builds reduzem tamanho de imagens
   - Imagens menores aceleram deploys e reduzem vulnerabilidades
   - Uso de usuário não-root aumenta segurança

2. **Versionamento Imutável**:
   - Tags específicas (semantic versioning, commit hash) são essenciais
   - Tag latest cria problemas de rastreabilidade e rollback
   - ImagePullPolicy deve ser explícito e adequado à tag

3. **Separação de Configuração**:
   - ConfigMaps para dados não sensíveis
   - Secrets para credenciais e tokens
   - envFrom simplifica injeção de múltiplas variáveis
   - Validação de variáveis previne falhas em runtime

4. **Segurança de Secrets**:
   - Base64 não é criptografia
   - Encryption at rest é necessário em produção
   - Secret managers externos (Vault) oferecem segurança superior
   - RBAC restritivo protege acessos

5. **Estratégias de Deployment**:
   - RollingUpdate é padrão e adequado para maioria
   - Blue-Green oferece rollback instantâneo
   - Canary reduz risco com exposição gradual
   - Escolha baseada em criticidade e recursos

6. **Rollback e Recuperação**:
   - Histórico de revisões permite reversão rápida
   - Rollback é simples com tags imutáveis
   - Monitoramento é essencial para detecção de problemas
   - revisionHistoryLimit balanceia rollback e consumo de recursos

### Melhores Práticas

1. **Sempre use tags específicas e imutáveis** em produção
2. **ConfigMaps e Secrets versionados** em Git (exceto valores sensíveis)
3. **Automatize build e push** via CI/CD com tags automáticas
4. **Documente variáveis esperadas** pela aplicação
5. **Teste rollbacks** em staging antes de produção
6. **Monitore métricas** durante e após deployments
7. **Use ferramentas** (Kustomize, Helm) para multi-ambiente
8. **Implemente smoke tests** após deployments
9. **Configure health checks** (liveness, readiness) adequados
10. **Mantenha imagens atualizadas** para patches de segurança

### Próximos Passos

Os conceitos apresentados preparam para tópicos avançados:

- **Health Checks Avançados**: Startup, liveness, readiness probes
- **Persistent Volumes**: Armazenamento de dados stateful
- **Horizontal Pod Autoscaler**: Escalabilidade automática
- **Ingress Controllers**: Roteamento HTTP/HTTPS avançado
- **Service Mesh**: Istio, Linkerd para observabilidade
- **GitOps**: ArgoCD, Flux para deployments declarativos
- **CI/CD Integration**: Jenkins, GitHub Actions, GitLab CI
- **Monitoramento**: Prometheus, Grafana, ELK stack

### Recomendações Finais

Para equipes em produção:

1. **Estabeleça pipeline de CI/CD** robusto e automatizado
2. **Implemente observabilidade** desde o início
3. **Documente processos** de deployment e rollback
4. **Realize post-mortems** de incidents para aprendizado
5. **Treine equipe** em procedimentos de emergência
6. **Mantenha staging** espelho de produção para testes
7. **Automatize testes** de integração e smoke tests
8. **Revise configurações** de segurança periodicamente

## 10. Referências Bibliográficas

### Documentação Oficial

- Kubernetes Documentation. "ConfigMaps". Disponível em: https://kubernetes.io/docs/concepts/configuration/configmap/. Acesso em: 2024.

- Kubernetes Documentation. "Secrets". Disponível em: https://kubernetes.io/docs/concepts/configuration/secret/. Acesso em: 2024.

- Kubernetes Documentation. "Managing Secrets using kubectl". Disponível em: https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/. Acesso em: 2024.

- Kubernetes Documentation. "Images". Disponível em: https://kubernetes.io/docs/concepts/containers/images/. Acesso em: 2024.

- Kubernetes Documentation. "Deployment Strategies". Disponível em: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy. Acesso em: 2024.

### Ferramentas

- Docker Documentation. "Best practices for writing Dockerfiles". Disponível em: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/. Acesso em: 2024.

- Kustomize Documentation. Disponível em: https://kustomize.io/. Acesso em: 2024.

- Helm Documentation. Disponível em: https://helm.sh/docs/. Acesso em: 2024.

- HashiCorp Vault Documentation. "Vault Agent with Kubernetes". Disponível em: https://www.vaultproject.io/docs/platform/k8s. Acesso em: 2024.

### Livros

- Burns, Brendan; Beda, Joe; Hightower, Kelsey. "Kubernetes: Up and Running". O'Reilly Media, 2019.

- Luksa, Marko. "Kubernetes in Action". Manning Publications, 2018.

- Hausenblas, Michael; Schimanski, Stefan. "Programming Kubernetes". O'Reilly Media, 2019.

### Artigos

- Kubernetes Blog. "Encrypting Secret Data at Rest". Disponível em: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/. Acesso em: 2024.

- Kubernetes Blog. "Deployment Strategies". Disponível em: https://kubernetes.io/blog/. Acesso em: 2024.

## 11. Apêndice

### Apêndice A: Comandos Úteis

#### ConfigMaps

```bash
# Criar
kubectl create configmap app-config --from-literal=KEY=VALUE
kubectl create configmap app-config --from-file=app.env
kubectl apply -f configmap.yaml

# Listar
kubectl get configmaps
kubectl get cm

# Visualizar
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml

# Editar
kubectl edit configmap app-config

# Deletar
kubectl delete configmap app-config
```

#### Secrets

```bash
# Criar
kubectl create secret generic db-creds --from-literal=password=secret
kubectl create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...
kubectl apply -f secret.yaml

# Listar
kubectl get secrets

# Visualizar (valores em base64)
kubectl get secret db-creds -o yaml

# Decodificar
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d

# Deletar
kubectl delete secret db-creds
```

#### Deployment e Rollout

```bash
# Aplicar Deployment
kubectl apply -f deployment.yaml

# Atualizar imagem
kubectl set image deployment/myapp app=myapp:2.0.0

# Status do rollout
kubectl rollout status deployment/myapp

# Histórico
kubectl rollout history deployment/myapp

# Rollback
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2

# Pausar/Retomar
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp

# Restart (força recreação de Pods)
kubectl rollout restart deployment/myapp
```

### Apêndice B: Exemplo Completo de Aplicação

#### Dockerfile

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package*.json ./
EXPOSE 3000
USER node
CMD ["node", "dist/main.js"]
```

#### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_NAME: "MyApplication"
  APP_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "3000"
```

#### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@postgres:5432/mydb"
  API_KEY: "sk_live_abc123def456"
  JWT_SECRET: "supersecret123"
```

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: "1.2.0"
    spec:
      containers:
      - name: app
        image: myregistry/myapp:1.2.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          protocol: TCP
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  revisionHistoryLimit: 10
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3000
```

### Apêndice C: Glossário e Termos Técnicos

**Base64**: Esquema de encoding que representa dados binários em formato ASCII. NÃO é criptografia.

**Blue-Green Deployment**: Estratégia com duas versões completas (blue e green), switch instantâneo de tráfego.

**Canary Deployment**: Release gradual para subconjunto de usuários, permitindo validação antes de rollout completo.

**Change-Cause**: Anotação que documenta razão de mudança em revisão de Deployment.

**ConfigMap**: Objeto Kubernetes para armazenar dados de configuração não sensíveis em pares chave-valor.

**Container Registry**: Repositório para armazenar e distribuir imagens Docker.

**Dockerfile**: Arquivo de texto com instruções para construir imagem Docker.

**Encryption at Rest**: Criptografia de dados armazenados em disco (etcd).

**envFrom**: Campo que injeta todas chaves de ConfigMap/Secret como variáveis de ambiente.

**ImagePullPolicy**: Política que define quando Kubernetes deve puxar imagem do registry.

**ImagePullSecret**: Secret contendo credenciais para pull de imagens de registry privado.

**Immutable**: Recurso que não pode ser modificado após criação, apenas substituído.

**Kustomize**: Ferramenta nativa do Kubernetes para customização de manifestos sem templates.

**Multi-stage Build**: Técnica Dockerfile com múltiplos estágios FROM, resultando em imagem final menor.

**Opaque Secret**: Tipo genérico de Secret para dados arbitrários.

**Recreate**: Estratégia de deployment que deleta todos Pods antes de criar novos.

**Registry Credentials**: Autenticação para acessar container registry privado.

**Revision**: Versão específica na história de rollout de Deployment.

**revisionHistoryLimit**: Número de ReplicaSets antigas mantidas para rollback.

**Rollback**: Reversão para versão anterior de Deployment.

**Rolling Update**: Estratégia de atualização gradual mantendo disponibilidade.

**Secret**: Objeto Kubernetes para armazenar dados sensíveis (senhas, tokens) em base64.

**Semantic Versioning**: Esquema de versionamento MAJOR.MINOR.PATCH.

**stringData**: Campo de Secret que aceita texto plano, convertido para base64 automaticamente.

**Vault**: HashiCorp Vault, ferramenta de gerenciamento de secrets com criptografia e controle de acesso.

**Zero Downtime**: Deploy sem interrupção de serviço para usuários.
