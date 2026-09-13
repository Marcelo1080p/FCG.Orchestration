# FCG.Orchestration

Orquestração da plataforma **FIAP Cloud Games (FCG)** — sobe toda a arquitetura de microsserviços com um único comando.

## Arquitetura

```
                       ┌──────────────────┐
                       │   FCG.UsersAPI    │ :5001
                       └────────┬─────────┘
                                │ UserCreatedEvent
                ┌───────────────┼───────────────────┐
                ▼               ▼                   │
     ┌──────────────────┐  ┌──────────────────────┐ │
     │  FCG.CatalogAPI  │  │ NotificationsFunction│ │
     │      :5002       │  │     (serverless)     │ │
     └────────┬─────────┘  └──────────▲───────────┘ │
              │ OrderPlacedEvent      │             │
              ▼                       │ PaymentProcessedEvent
     ┌──────────────────┐             │
     │ FCG.PaymentsAPI  │─────────────┘
     │      :5003       │
     └──────────────────┘

     Mensageria: RabbitMQ (:5672, management :15672)
     Banco de dados: SQL Server (:1434)
```

## Repositórios

| Serviço | Responsabilidade |
|---|---|
| [FCG.UsersAPI](https://github.com/Marcelo1080p/FCG.UsersAPI) | Usuários e autenticação JWT |
| [FCG.CatalogAPI](https://github.com/Marcelo1080p/FCG.CatalogAPI) | Catálogo de jogos e aquisições |
| [FCG.PaymentsAPI](https://github.com/Marcelo1080p/FCG.PaymentsAPI) | Processamento de pagamentos |
| [FCG.NotificationsFunction](https://github.com/Marcelo1080p/FCG.NotificationsFunction) | Notificações como função serverless |

## Como executar

Pré-requisito: Docker Desktop. Os 4 repositórios devem estar clonados como irmãos desta pasta:

```
FCG/
├── FCG.Orchestration/   (este repositório)
├── FCG.UsersAPI/
├── FCG.CatalogAPI/
├── FCG.PaymentsAPI/
└── FCG.NotificationsAPI/
```

```bash
docker compose up -d --build
```

| Serviço | URL |
|---|---|
| UsersAPI (Swagger) | http://localhost:5001/swagger |
| CatalogAPI (Swagger) | http://localhost:5002/swagger |
| PaymentsAPI (Swagger) | http://localhost:5003/swagger |
| NotificationsFunction | sem porta — acionada pelas filas |
| RabbitMQ Management | http://localhost:15672 (guest/guest) |

## Fluxo de teste ponta a ponta

1. **Login como admin** — `POST /api/auth/login` no UsersAPI (`admin@fcg.com` / `Admin@123`)
2. **Cadastrar um jogo** — `POST /api/games` no CatalogAPI com o token de admin
3. **Registrar um usuário** — `POST /api/auth/register` no UsersAPI
   - NotificationsAPI loga a mensagem de boas-vindas (`UserCreatedEvent`)
4. **Login com o novo usuário** e **adquirir o jogo** — `POST /api/games/{id}/acquire`
   - CatalogAPI publica `OrderPlacedEvent`
   - PaymentsAPI processa o pagamento e publica `PaymentProcessedEvent`
   - NotificationsAPI loga a confirmação da compra
5. **Conferir o pagamento** — `GET /api/payments` no PaymentsAPI com token de admin

Logs dos consumidores:

```bash
docker compose logs -f notificationsapi paymentsapi catalogapi
```

## Arquitetura serverless

O antigo microsserviço `FCG.NotificationsAPI` foi substituído por uma **função serverless**, no repositório [FCG.NotificationsFunction](https://github.com/Marcelo1080p/FCG.NotificationsFunction).

O serviço passava a maior parte do tempo ocioso, apenas aguardando eventos esporádicos — manter um container de pé 24 horas por dia para isso desperdiçava recursos. Como função, o código só executa quando chega mensagem na fila.

| Função | Fila que dispara | Ação |
|---|---|---|
| `UserCreatedFunction` | `notifications-user-created` | Boas-vindas ao novo usuário |
| `PaymentProcessedFunction` | `notifications-payment-processed` | Confirmação da compra |

A função é **Azure Functions v4** (.NET 8, isolated worker), acionada diretamente pelas filas do RabbitMQ — sem endpoint HTTP. Localmente ela roda no runtime oficial do Azure Functions em container, acompanhada do **Azurite**, emulador de storage exigido pelo runtime. A infraestrutura de produção está descrita em Bicep, no diretório `infra/` daquele repositório, usando plano de consumo (`Y1`).

Para acompanhar o acionamento:

```bash
docker compose logs -f notificationsfunction
```

## API Gateway

**Kong** em modo declarativo (DB-less) é a **porta de entrada única** da plataforma. Toda requisição externa passa por ele, que valida o token JWT e roteia para o microsserviço correspondente. A configuração fica versionada em `gateway/kong.yml`.

### Rotas

| Rota | Destino | Métodos | Token |
|---|---|---|---|
| `/api/auth` | UsersAPI | Todos | Não — é aqui que o token é obtido |
| `/api/users` | UsersAPI | Todos | **Sim** |
| `/api/games` | CatalogAPI | GET | Não — catálogo e avaliações são públicos |
| `/api/games` | CatalogAPI | POST, PUT, PATCH, DELETE | **Sim** |

### Validação de JWT no gateway

O Kong valida a assinatura dos tokens emitidos pelo UsersAPI antes de encaminhar a requisição. O `consumer` declarado usa como chave a claim `iss` do token (`FCG.UsersAPI`) e o mesmo segredo HMAC da aplicação.

Requisições sem token, ou com token inválido ou expirado, recebem `401` **do próprio gateway** — o tráfego não chega ao microsserviço. Dá para confirmar pelo cabeçalho da resposta: `Server: kong/3.7.1` quando o bloqueio é do gateway, contra `Server: Kestrel` quando vem da aplicação.

Os microsserviços continuam validando o token por conta própria, em defesa em profundidade: mesmo que alguém alcance o serviço sem passar pelo gateway, a autorização por papel (`Admin`) continua valendo.

### Acesso

| Porta | Finalidade |
|---|---|
| 8000 | Proxy — ponto de entrada da plataforma |
| 8001 | Admin API do Kong (inspecionar rotas e serviços) |

Exemplo de uso ponta a ponta pelo gateway:

```bash
curl -X POST http://localhost:8000/api/auth/login -H "Content-Type: application/json" -d "{\"email\":\"admin@fcg.com\",\"password\":\"Admin@123\"}"
```

### Rodando os serviços fora do Docker

Assim como no Prometheus, `gateway/kong.yml` aponta para os serviços pelo DNS interno. Use `gateway/kong.local.yml` quando os microsserviços estiverem rodando na máquina via `dotnet run`.

## Observabilidade

**Stack escolhida: Opção A — Prometheus + Grafana** (código aberto), implantada por manifests Kubernetes.

A opção por Prometheus/Grafana em vez de uma plataforma de APM gerenciada (Datadog ou New Relic) se deu por não exigir conta externa nem chave de API, e por permitir que toda a stack seja versionada e implantada junto com a aplicação — inclusive o dashboard, provisionado como código.

### Instrumentação

`UsersAPI` e `CatalogAPI` usam a biblioteca `prometheus-net.AspNetCore`, que expõe o endpoint `/metrics` e coleta automaticamente, a cada requisição HTTP, a duração, o total e o status code — com rótulos de controller, action e endpoint.

### Dashboard

O dashboard `FCG — Visão Geral dos Microsserviços` é provisionado automaticamente ao subir o Grafana e traz:

| Painel | Métrica |
|---|---|
| Requisições por segundo | Throughput agregado |
| Latência p95 | Percentil 95 do tempo de resposta |
| Taxa de erro | Percentual de respostas 5xx |
| Throughput por serviço | Requisições/s por microsserviço |
| Latência p95 por serviço | Comparativo entre UsersAPI e CatalogAPI |
| Requisições por status HTTP | Volume por código de resposta |
| Latência p95 por endpoint | Detalhamento por controller/action |
| Requisições com falha | Séries de 4xx e 5xx por serviço |

### Acesso

| Ferramenta | URL | Credenciais |
|---|---|---|
| Prometheus | http://localhost:9090 | — |
| Grafana | http://localhost:3000 | admin / admin |

### Rodando os serviços fora do Docker

O arquivo `observability/prometheus/prometheus.yml` aponta para os serviços pelo nome no DNS interno (docker-compose ou Kubernetes). Se os microsserviços estiverem rodando direto na máquina via `dotnet run`, use `observability/prometheus/prometheus.local.yml`, que aponta para `host.docker.internal` nas portas locais.

## Kubernetes

Manifests da infraestrutura (RabbitMQ e SQL Server) em `k8s/`. Cada microsserviço tem seus próprios manifests no diretório `k8s/` do respectivo repositório.

```bash
kubectl apply -f k8s/
kubectl apply -f ../FCG.UsersAPI/k8s/
kubectl apply -f ../FCG.CatalogAPI/k8s/
kubectl apply -f ../FCG.PaymentsAPI/k8s/
kubectl apply -f ../FCG.NotificationsAPI/k8s/
```
