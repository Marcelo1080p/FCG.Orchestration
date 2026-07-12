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
     │  FCG.CatalogAPI  │  │ FCG.NotificationsAPI │ │
     │      :5002       │  │        :5004         │ │
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
| [FCG.NotificationsAPI](https://github.com/Marcelo1080p/FCG.NotificationsAPI) | Notificações orientadas a eventos |

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
| NotificationsAPI (health) | http://localhost:5004/health |
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

## Kubernetes

Manifests da infraestrutura (RabbitMQ e SQL Server) em `k8s/`. Cada microsserviço tem seus próprios manifests no diretório `k8s/` do respectivo repositório.

```bash
kubectl apply -f k8s/
kubectl apply -f ../FCG.UsersAPI/k8s/
kubectl apply -f ../FCG.CatalogAPI/k8s/
kubectl apply -f ../FCG.PaymentsAPI/k8s/
kubectl apply -f ../FCG.NotificationsAPI/k8s/
```
