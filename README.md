# ToggleMaster — Guia de Execução

ToggleMaster é uma plataforma de **feature flags** construída como uma arquitetura de microsserviços. Ela permite criar, gerenciar e avaliar flags de funcionalidade com regras de segmentação sofisticadas (ex: porcentagem de usuários, país, ID), além de registrar eventos de analytics via AWS.

---

**Serviços e Portas:**

| Serviço              | Tecnologia | Porta | Banco de Dados       |
|----------------------|------------|-------|----------------------|
| `auth-service`       | Go         | 8001  | PostgreSQL (porta 5432) |
| `flag-service`       | Python     | 8002  | PostgreSQL (porta 5434) |
| `targeting-service`  | Python     | 8003  | PostgreSQL (porta 5433) |
| `evaluation-service` | Go         | 8004  | Redis (porta 6379)   |
| `analytics-service`  | Python     | 8005  | AWS DynamoDB         |

---

## Pré-requisitos

Antes de começar, você precisa ter instalado:

- [Docker](https://docs.docker.com/get-docker/) e [Docker Compose](https://docs.docker.com/compose/install/) (versão 2+)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configurado com credenciais válidas
- Acesso a uma conta AWS com permissões para criar recursos SQS e DynamoDB

> **Por que Docker?** Todo o ambiente local — bancos de dados, cache e microsserviços — sobe com um único comando, sem precisar instalar PostgreSQL, Redis ou Go/Python na sua máquina.

---

## Infraestrutura AWS (Pré-requisito obrigatório)

Os serviços `evaluation-service` e `analytics-service` dependem de recursos AWS que precisam existir **antes** de subir os containers. Crie-os com os comandos abaixo.

### 1. Fila SQS

O `evaluation-service` publica eventos de decisão nessa fila. O `analytics-service` consome esses eventos.

```bash
aws sqs create-queue --queue-name togglemaster-events --region us-east-1
```

Após executar, o comando retorna a URL da fila. Anote essa URL — ela será usada nas variáveis de ambiente.

> **Por que isso vem primeiro?** Sem a fila SQS existindo, o `evaluation-service` não consegue publicar eventos e o `analytics-service` não tem de onde consumir. Criar antes garante que a URL já estará disponível quando você configurar o `.env`.

### 2. Tabela DynamoDB

O `analytics-service` grava cada evento de avaliação nessa tabela.

```bash
aws dynamodb create-table \
    --table-name ToggleMasterAnalytics \
    --attribute-definitions AttributeName=event_id,AttributeType=S \
    --key-schema AttributeName=event_id,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
    --region us-east-1
```

> **Por que throughput mínimo?** `ReadCapacityUnits=1,WriteCapacityUnits=1` é suficiente para testes e é coberto pelo free tier da AWS. Em produção, considere o modo `PAY_PER_REQUEST`.

---

## Configuração do Arquivo `.env`

O `docker-compose.yaml` lê todas as variáveis de ambiente de um único arquivo `.env` na raiz do projeto. Crie esse arquivo antes de subir os serviços.

> **Por que um único `.env` na raiz?** O Docker Compose, por padrão, carrega o arquivo `.env` do mesmo diretório onde é executado, injetando as variáveis em todos os serviços. Isso centraliza a configuração em um só lugar.

Crie o arquivo `.env` na raiz do projeto com o seguinte conteúdo e preencha cada valor:

```dotenv
# ==============================================================================
# auth-service — Banco de Dados
# ==============================================================================
# Usuário, senha e nome do banco para o PostgreSQL do auth-service.
# Esses valores são usados TANTO pelo container do banco quanto pelo serviço Go.
DB_USERNAME=
DB_PASSWORD=
DB_DATABASE=

# ==============================================================================
# auth-service — Chave Mestra
# ==============================================================================
# Chave usada para proteger o endpoint de criação de API Keys (/admin/keys).
# Deve ser uma string longa e aleatória. Guarde-a em um cofre de senhas.
MASTER_KEY=

# ==============================================================================
# flag-service — Banco de Dados
# ==============================================================================
FLAG_DB_USERNAME=
FLAG_DB_PASSWORD=
FLAG_DB_DATABASE=

# ==============================================================================
# targeting-service — Banco de Dados
# ==============================================================================
TARGETING_DB_USERNAME=
TARGETING_DB_PASSWORD=
TARGETING_DB_DATABASE=

# ==============================================================================
# evaluation-service — Chave de API de Serviço
# ==============================================================================
# API Key que o evaluation-service usa para autenticar chamadas internas
# ao flag-service e ao targeting-service.
# IMPORTANTE: essa chave só pode ser gerada DEPOIS que o auth-service estiver
# rodando. Veja a seção "Primeira Execução" abaixo.
SERVICE_API_KEY=

# ==============================================================================
# URLs internas entre serviços
# ==============================================================================
# O evaluation-service usa essas URLs para chamar os outros serviços.
# Os valores abaixo funcionam dentro da rede Docker Compose.
AUTH_SERVICE_URL=http://auth-service:8001
FLAG_SERVICE_URL=http://flag-service:8002
TARGETING_SERVICE_URL=http://targeting-service:8003

# ==============================================================================
# Redis
# ==============================================================================
# URL do Redis usado como cache pelo evaluation-service.
# O valor abaixo aponta para o container Redis definido no docker-compose.
REDIS_URL=redis://redis:6379

# ==============================================================================
# AWS
# ==============================================================================
# URL completa da fila SQS criada no passo anterior.
# Exemplo: https://sqs.us-east-1.amazonaws.com/123456789012/togglemaster-events
AWS_SQS_URL=

# Nome da tabela DynamoDB criada no passo anterior.
AWS_DYNAMODB_TABLE=ToggleMasterAnalytics

# Região AWS onde seus recursos foram criados.
AWS_REGION=us-east-1

# Credenciais de acesso à AWS.
# Se estiver usando AWS Academy, adicione também AWS_SESSION_TOKEN.
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

> **Segurança:** Nunca comite o arquivo `.env` no repositório. Adicione-o ao `.gitignore`.

---

## Primeira Execução — Obtendo a `SERVICE_API_KEY`

O `evaluation-service` precisa de uma API Key válida para se autenticar nos outros serviços, mas essa chave só pode ser gerada após o `auth-service` estar rodando. Por isso, a primeira execução é feita em duas etapas.

### Etapa 1 — Suba apenas o auth-service e seus dependentes

```bash
docker compose up -d db-auth-service auth-service
```

Aguarde alguns segundos e confirme que o serviço está saudável:

```bash
curl http://localhost:8001/health
# Saída esperada: {"status":"ok"}
```

### Etapa 2 — Gere a SERVICE_API_KEY

Use a `MASTER_KEY` que você definiu no `.env` para criar a chave interna do `evaluation-service`:

```bash
curl -X POST http://localhost:8001/admin/keys \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <sua_MASTER_KEY>" \
  -d '{"name": "evaluation-service-key"}'
```

A resposta será parecida com:

```json
{
  "name": "evaluation-service-key",
  "key": "tm_key_xxxxxxxxxxxxxxxx",
  "message": "Guarde esta chave com segurança! Você não poderá vê-la novamente."
}
```

> **Por que só aparece uma vez?** O `auth-service` armazena apenas o hash da chave, não o valor em texto puro. Se perder a chave, precisará gerar uma nova.

### Etapa 3 — Atualize o `.env` com a chave gerada

Abra o `.env` e preencha:

```dotenv
SERVICE_API_KEY=tm_key_xxxxxxxxxxxxxxxx
```

### Etapa 4 — Suba todos os serviços

```bash
docker compose up -d
```

---

## Subindo o Ambiente Completo

Com o `.env` preenchido corretamente e a infraestrutura AWS criada, suba todos os serviços:

```bash
docker compose up -d
```

Verifique se todos os containers estão rodando:

```bash
docker compose ps
```

Confira a saúde de cada serviço:

```bash
curl http://localhost:8001/health   # auth-service
curl http://localhost:8002/health   # flag-service
curl http://localhost:8003/health   # targeting-service
curl http://localhost:8004/health   # evaluation-service
curl http://localhost:8005/health   # analytics-service
```

Todos devem retornar `{"status":"ok"}`.

---

## Microsserviços

---

### auth-service (Go — Porta 8001)

Responsável por criar e validar API Keys. **Todos os outros serviços dependem dele** para autenticar requisições — nenhuma flag, regra ou avaliação funciona sem uma chave válida.

#### Como funciona

- O endpoint `/admin/keys` é protegido pela `MASTER_KEY` e cria novas chaves.
- O endpoint `/validate` é chamado internamente pelos outros serviços para verificar se a chave do cliente é válida.
- As chaves são armazenadas como hash no PostgreSQL, nunca em texto puro.

#### Variáveis de ambiente

| Variável       | Descrição                                                  |
|----------------|------------------------------------------------------------|
| `DATABASE_URL` | String de conexão com o PostgreSQL do auth-service         |
| `PORT`         | Porta do serviço (definida como `8001` no docker-compose)  |
| `MASTER_KEY`   | Chave que protege a criação de novas API Keys              |

#### Endpoints principais

```bash
# Health check
GET  /health

# Cria uma nova API Key (requer MASTER_KEY no header Authorization)
POST /admin/keys
Body: { "name": "nome-da-chave" }

# Valida uma API Key (usado internamente pelos outros serviços)
GET  /validate
Header: Authorization: Bearer <api_key>
```

---

### flag-service (Python — Porta 8002)

Gerencia as **definições** das feature flags (nome, descrição, status ativo/inativo). É o CRUD central da plataforma.

#### Como funciona

- Todas as rotas (exceto `/health`) exigem uma API Key válida no header `Authorization`.
- A cada requisição, o serviço chama o `auth-service` para validar a chave antes de processar.
- Os dados são persistidos no PostgreSQL dedicado ao serviço.

> **Por que um banco separado por serviço?** Isolamento de dados — um problema no banco do `flag-service` não afeta o `auth-service` ou o `targeting-service`. Cada serviço é dono dos seus próprios dados.

#### Variáveis de ambiente

| Variável           | Descrição                                                    |
|--------------------|--------------------------------------------------------------|
| `DATABASE_URL`     | String de conexão com o PostgreSQL do flag-service           |
| `PORT`             | Porta do serviço (definida como `8002` no docker-compose)    |
| `AUTH_SERVICE_URL` | URL interna do auth-service para validação de chaves         |

#### Endpoints principais

```bash
# Health check
GET  /health

# Lista todas as flags
GET  /flags

# Cria uma nova flag
POST /flags
Body: { "name": "nome-da-flag", "description": "...", "is_enabled": true }

# Busca uma flag pelo nome
GET  /flags/<flag_name>

# Atualiza uma flag
PUT  /flags/<flag_name>

# Remove uma flag
DELETE /flags/<flag_name>
```

---

### targeting-service (Python — Porta 8003)

Gerencia as **regras de segmentação** associadas a cada flag. Define critérios como "50% dos usuários", "país = Brasil" ou "lista de IDs específicos".

#### Como funciona

- Assim como o `flag-service`, todas as rotas exigem uma API Key válida.
- As regras são armazenadas como JSON no PostgreSQL, permitindo estruturas flexíveis.
- O `evaluation-service` consulta este serviço para saber como avaliar uma flag para um usuário específico.

#### Variáveis de ambiente

| Variável           | Descrição                                                        |
|--------------------|------------------------------------------------------------------|
| `DATABASE_URL`     | String de conexão com o PostgreSQL do targeting-service          |
| `PORT`             | Porta do serviço (definida como `8003` no docker-compose)        |
| `AUTH_SERVICE_URL` | URL interna do auth-service para validação de chaves             |

#### Endpoints principais

```bash
# Health check
GET  /health

# Cria uma regra de segmentação para uma flag
POST /rules
Body: { "flag_name": "nome-da-flag", "is_enabled": true, "rules": { "type": "PERCENTAGE", "value": 50 } }

# Busca a regra de uma flag
GET  /rules/<flag_name>

# Atualiza a regra de uma flag
PUT  /rules/<flag_name>

# Remove a regra de uma flag
DELETE /rules/<flag_name>
```

**Tipos de regra suportados:**

| Tipo         | Exemplo de payload `rules`                          |
|--------------|------------------------------------------------------|
| `PERCENTAGE` | `{ "type": "PERCENTAGE", "value": 50 }`              |
| `USER_LIST`  | `{ "type": "USER_LIST", "ids": ["user-1", "user-2"] }` |

---

### evaluation-service (Go — Porta 8004)

O "caminho quente" da plataforma. É o único endpoint que aplicações externas chamam para saber se uma flag está ativa para um determinado usuário.

#### Como funciona

1. Recebe `GET /evaluate?user_id=...&flag_name=...`
2. Consulta o **Redis** (cache). Se a resposta estiver em cache (**Cache HIT**), retorna imediatamente.
3. Se não estiver (**Cache MISS**), busca a definição no `flag-service` e a regra no `targeting-service`, avalia e armazena no Redis com TTL curto.
4. Retorna `true` ou `false` para o cliente.
5. Envia **assincronamente** um evento para a fila **AWS SQS** para registro de analytics.

> **Por que Redis?** A avaliação de flags pode ser chamada milhares de vezes por segundo. O Redis evita chamadas repetidas ao `flag-service` e `targeting-service`, reduzindo latência e carga nos demais serviços.

> **Por que SQS assíncrono?** Registrar analytics não deve atrasar a resposta para o cliente. Publicar na fila é quase instantâneo; o processamento real acontece no `analytics-service` de forma desacoplada.

#### Variáveis de ambiente

| Variável                | Descrição                                                              |
|-------------------------|------------------------------------------------------------------------|
| `PORT`                  | Porta do serviço (definida como `8004` no docker-compose)              |
| `REDIS_URL`             | URL de conexão com o Redis                                             |
| `FLAG_SERVICE_URL`      | URL interna do flag-service                                            |
| `TARGETING_SERVICE_URL` | URL interna do targeting-service                                       |
| `SERVICE_API_KEY`       | API Key para autenticar chamadas ao flag-service e targeting-service   |
| `AWS_SQS_URL`           | URL completa da fila SQS para publicação de eventos                    |
| `AWS_REGION`            | Região AWS da fila SQS                                                 |
| `AWS_ACCESS_KEY_ID`     | Credencial AWS                                                         |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS                                                         |

#### Endpoints principais

```bash
# Health check
GET  /health

# Avalia uma flag para um usuário
GET  /evaluate?user_id=<id>&flag_name=<nome>
# Retorna: { "flag_name": "...", "user_id": "...", "result": true }
```

---

### analytics-service (Python — Porta 8005)

Worker de backend que consome eventos da fila SQS e os persiste no DynamoDB para análise histórica.

#### Como funciona

- Não possui API pública (apenas `/health`).
- Em background, um worker consome continuamente a fila SQS.
- Cada mensagem representa uma decisão de avaliação e é gravada como um item no DynamoDB.

> **Por que DynamoDB?** É um banco NoSQL serverless da AWS, ideal para grandes volumes de eventos com estrutura variável. Não requer provisionamento de servidor e escala automaticamente.

#### Recursos AWS necessários

| Recurso   | Nome                    | Chave primária               |
|-----------|-------------------------|------------------------------|
| SQS       | `togglemaster-events`   | —                            |
| DynamoDB  | `ToggleMasterAnalytics` | `event_id` (String, HASH)    |

#### Variáveis de ambiente

| Variável                | Descrição                                              |
|-------------------------|--------------------------------------------------------|
| `PORT`                  | Porta do health check (definida como `8005`)           |
| `AWS_SQS_URL`           | URL completa da fila SQS                               |
| `AWS_DYNAMODB_TABLE`    | Nome da tabela DynamoDB (`ToggleMasterAnalytics`)      |
| `AWS_REGION`            | Região AWS dos recursos                                |
| `AWS_ACCESS_KEY_ID`     | Credencial AWS                                         |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS                                         |

---

## Fluxo Completo de Uso

Após todo o ambiente estar rodando, siga este fluxo para testar o ciclo completo:

### 1. Crie uma API Key para uso geral

```bash
curl -X POST http://localhost:8001/admin/keys \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <sua_MASTER_KEY>" \
  -d '{"name": "minha-aplicacao"}'
```

Guarde o valor de `key` retornado — use-o como `<API_KEY>` nos próximos passos.

### 2. Crie uma feature flag

```bash
curl -X POST http://localhost:8002/flags \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{"name": "novo-dashboard", "description": "Ativa o novo dashboard", "is_enabled": true}'
```

### 3. Crie uma regra de segmentação

```bash
curl -X POST http://localhost:8003/rules \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{"flag_name": "novo-dashboard", "is_enabled": true, "rules": {"type": "PERCENTAGE", "value": 50}}'
```

### 4. Avalie a flag para diferentes usuários

```bash
curl "http://localhost:8004/evaluate?user_id=user-001&flag_name=novo-dashboard"
curl "http://localhost:8004/evaluate?user_id=user-002&flag_name=novo-dashboard"
curl "http://localhost:8004/evaluate?user_id=user-003&flag_name=novo-dashboard"
```

Aproximadamente 50% dos usuários receberão `"result": true`.

### 5. Verifique os eventos no DynamoDB

```bash
aws dynamodb scan --table-name ToggleMasterAnalytics --region us-east-1
```

---

## Comandos Úteis

```bash
# Ver logs de um serviço específico
docker compose logs -f evaluation-service

# Parar todos os containers sem remover volumes
docker compose stop

# Remover todos os containers e volumes (limpa o banco de dados)
docker compose down -v

# Rebuild de um serviço após mudança de código
docker compose up -d --build <nome-do-servico>
```

---

## Solução de Problemas

| Problema                                        | Causa provável                                            | Solução                                                               |
|-------------------------------------------------|-----------------------------------------------------------|-----------------------------------------------------------------------|
| `auth-service` não sobe                         | PostgreSQL ainda inicializando                            | Aguarde 5–10 segundos e execute `docker compose up -d` novamente      |
| `Chave de API inválida ou inativa`              | `SERVICE_API_KEY` não preenchida ou chave inválida        | Gere uma nova chave no `auth-service` e atualize o `.env`             |
| `evaluation-service` não publica no SQS         | Credenciais AWS incorretas ou URL da fila errada          | Verifique `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` e `AWS_SQS_URL` |
| `analytics-service` não grava no DynamoDB       | Tabela não criada ou credenciais sem permissão            | Crie a tabela conforme a seção de infraestrutura AWS                  |
| Cache não funciona                              | Redis não acessível                                       | Confirme que o container `redis` está rodando com `docker compose ps` |
