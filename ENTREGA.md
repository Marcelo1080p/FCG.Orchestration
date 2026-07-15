# Tech Challenge — Fase 2 | FIAP Cloud Games (FCG)

## 📦 Repositórios

| Repositório | Responsabilidade |
|---|---|
| [FCG.UsersAPI](https://github.com/Marcelo1080p/FCG.UsersAPI) | Cadastro de usuários, autenticação JWT, gerenciamento admin |
| [FCG.CatalogAPI](https://github.com/Marcelo1080p/FCG.CatalogAPI) | Catálogo de jogos, descontos e aquisição |
| [FCG.PaymentsAPI](https://github.com/Marcelo1080p/FCG.PaymentsAPI) | Processamento de pagamentos orientado a eventos |
| [FCG.NotificationsAPI](https://github.com/Marcelo1080p/FCG.NotificationsAPI) | Notificações (boas-vindas e confirmação de compra) |
| [FCG.Orchestration](https://github.com/Marcelo1080p/FCG.Orchestration) | docker-compose e manifests de infraestrutura (este repositório) |

Os testes unitários de cada serviço estão na branch `feature/testes-unitarios` do respectivo repositório (56 testes no total: 24 Users, 22 Catalog, 10 Payments).

## 🏗️ Arquitetura

O monólito da Fase 1 foi decomposto em 4 microsserviços independentes, cada um com seu próprio repositório, banco de dados e ciclo de deploy:

```
                       ┌──────────────────┐
                       │   FCG.UsersAPI    │
                       └────────┬─────────┘
                                │ UserCreatedEvent
                ┌───────────────┼───────────────────┐
                ▼               ▼                   │
     ┌──────────────────┐  ┌──────────────────────┐ │
     │  FCG.CatalogAPI  │  │ FCG.NotificationsAPI │ │
     └────────┬─────────┘  └──────────▲───────────┘ │
              │ OrderPlacedEvent      │             │
              ▼                       │ PaymentProcessedEvent
     ┌──────────────────┐             │
     │ FCG.PaymentsAPI  │─────────────┘
     └──────────────────┘
```

### Decisões técnicas

- **Clean Architecture / DDD** — camadas Domain, Application, Infrastructure e API em cada serviço
- **CQRS com MediatR** — commands e queries segregados
- **Mensageria com MassTransit + RabbitMQ** — comunicação assíncrona entre serviços; contratos de eventos compartilhados no namespace `FCG.Contracts.Events`
- **Database per service** — FCG_UsersDB, FCG_CatalogDB e FCG_PaymentsDB isolados (NotificationsAPI é stateless)
- **JWT compartilhado** — tokens emitidos pelo UsersAPI e validados pelos demais serviços (mesmo secret/issuer/audience)
- **Idempotência** — PaymentsAPI ignora pedidos já processados (índice único por OrderId)
- **Resiliência** — se um consumidor estiver fora do ar, as mensagens permanecem na fila e são processadas quando ele retornar

### Stack

.NET 8 · ASP.NET Core · EF Core 8 · SQL Server · RabbitMQ · MassTransit · MediatR · BCrypt · xUnit · NSubstitute · Docker (multi-stage) · Kubernetes

## 🚀 Como executar

Pré-requisito: Docker Desktop. Clone os 5 repositórios como pastas irmãs e rode:

```bash
cd FCG.Orchestration
docker compose up -d --build
```

| Serviço | URL |
|---|---|
| UsersAPI | http://localhost:5001/swagger |
| CatalogAPI | http://localhost:5002/swagger |
| PaymentsAPI | http://localhost:5003/swagger |
| NotificationsAPI | http://localhost:5004/health |
| RabbitMQ Management | http://localhost:15672 (guest/guest) |

Admin criado automaticamente: `admin@fcg.com` / `Admin@123`

O fluxo de teste ponta a ponta está detalhado no [README](README.md#fluxo-de-teste-ponta-a-ponta).

### Kubernetes

Cada repositório possui o diretório `k8s/` com Deployment, Service, ConfigMap e Secret. A infraestrutura (RabbitMQ e SQL Server) está em [`k8s/`](k8s/) deste repositório.

## ✅ Requisitos da fase atendidos

| Requisito | Implementação |
|---|---|
| Decomposição em microsserviços | 4 serviços independentes, um repositório cada |
| Comunicação assíncrona orientada a eventos | RabbitMQ + MassTransit (3 eventos, 4 filas) |
| Banco de dados por serviço | 3 bancos SQL Server isolados |
| Autenticação e autorização | JWT com roles (User/Admin) validado em todos os serviços |
| Containerização | Dockerfile multi-stage em cada serviço |
| Orquestração | docker-compose completo + manifests Kubernetes |
| Testes unitários | 56 testes (xUnit + NSubstitute) nas branches `feature/testes-unitarios` |
| Documentação | README em cada repositório com endpoints, eventos e variáveis de ambiente |

## 👤 Autor

**Marcelo** — [GitHub](https://github.com/Marcelo1080p)
