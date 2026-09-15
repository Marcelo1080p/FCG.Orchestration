# Diagramas da arquitetura

---

## 1. Evolução do projeto

Abertura da apresentação: de onde viemos e onde chegamos.

```mermaid
flowchart LR
    subgraph F1["Fase 1 — Monolito"]
        M["FCG.Host<br/>API unica<br/>SQL Server"]
    end

    subgraph F2["Fase 2 — Microsservicos"]
        U2["UsersAPI"]
        C2["CatalogAPI"]
        P2["PaymentsAPI"]
        N2["NotificationsAPI"]
        R2["RabbitMQ"]
    end

    subgraph F3["Fase 3 — Plataforma"]
        K3["Kong<br/>API Gateway"]
        S3["Function<br/>serverless"]
        O3["Prometheus<br/>Grafana"]
        D3["MongoDB<br/>Redis"]
    end

    F1 -->|"decomposicao"| F2
    F2 -->|"profissionalizacao"| F3
```

---

## 2. Visão geral da arquitetura

O diagrama principal: mostra o gateway como entrada única, a mensageria no centro e cada serviço com sua persistência.

```mermaid
flowchart TB
    Cliente(["Cliente"])

    Kong["Kong API Gateway<br/>porta 8000<br/>valida JWT"]

    Users["UsersAPI<br/>autenticacao"]
    Catalog["CatalogAPI<br/>catalogo e avaliacoes"]
    Payments["PaymentsAPI<br/>pagamentos"]
    Function["NotificationsFunction<br/>serverless"]

    Rabbit{{"RabbitMQ<br/>message broker"}}

    UsersDB[("SQL Server<br/>FCG_UsersDB")]
    CatalogDB[("SQL Server<br/>FCG_CatalogDB")]
    PaymentsDB[("SQL Server<br/>FCG_PaymentsDB")]
    Mongo[("MongoDB<br/>avaliacoes")]
    Redis[("Redis<br/>cache")]

    Prometheus["Prometheus"]
    Grafana["Grafana<br/>dashboard"]

    Cliente -->|"HTTP"| Kong
    Kong -->|"/api/auth<br/>/api/users"| Users
    Kong -->|"/api/games"| Catalog

    Users --> UsersDB
    Catalog --> CatalogDB
    Catalog --> Mongo
    Catalog --> Redis
    Payments --> PaymentsDB

    Users -->|"UserCreatedEvent"| Rabbit
    Catalog -->|"OrderPlacedEvent"| Rabbit
    Payments -->|"PaymentProcessedEvent"| Rabbit

    Rabbit -->|"catalog-user-created"| Catalog
    Rabbit -->|"payments-order-placed"| Payments
    Rabbit -->|"notifications-*"| Function

    Users -.->|"/metrics"| Prometheus
    Catalog -.->|"/metrics"| Prometheus
    Prometheus --> Grafana
```

---

## 3. Fluxo de cadastro de usuário

```mermaid
sequenceDiagram
    actor Cliente
    participant Kong as Kong Gateway
    participant Users as UsersAPI
    participant Rabbit as RabbitMQ
    participant Catalog as CatalogAPI
    participant Func as NotificationsFunction

    Cliente->>Kong: POST /api/auth/register
    Note over Kong: rota publica, sem token
    Kong->>Users: encaminha
    Users->>Users: valida e-mail unico<br/>gera hash BCrypt
    Users->>Rabbit: publica UserCreatedEvent
    Users-->>Cliente: 201 Created

    Note over Rabbit: exchange fanout<br/>entrega em paralelo
    Rabbit->>Catalog: fila catalog-user-created
    Rabbit->>Func: fila notifications-user-created
    Func->>Func: envia boas-vindas
```

---

## 4. Fluxo de compra

O fluxo mais completo — mostra a cadeia de eventos entre três serviços.

```mermaid
sequenceDiagram
    actor Cliente
    participant Kong as Kong Gateway
    participant Catalog as CatalogAPI
    participant Rabbit as RabbitMQ
    participant Pay as PaymentsAPI
    participant Func as NotificationsFunction

    Cliente->>Kong: POST /api/games/{id}/acquire
    Kong->>Kong: valida assinatura do JWT
    Kong->>Catalog: encaminha
    Catalog->>Catalog: verifica jogo ativo<br/>e se ja possui
    Catalog->>Rabbit: publica OrderPlacedEvent
    Catalog-->>Cliente: 200 OK (orderId)

    Note over Cliente,Catalog: resposta imediata:<br/>o pagamento segue assincrono

    Rabbit->>Pay: fila payments-order-placed
    Pay->>Pay: verifica idempotencia<br/>aprova pagamento
    Pay->>Rabbit: publica PaymentProcessedEvent
    Rabbit->>Func: fila notifications-payment-processed
    Func->>Func: notifica compra confirmada
```

---

## 5. Persistência poliglota

Uma única requisição usando as três tecnologias de armazenamento.

```mermaid
sequenceDiagram
    actor Cliente
    participant Catalog as CatalogAPI
    participant Redis
    participant SQL as SQL Server
    participant Mongo as MongoDB

    Cliente->>Catalog: GET /api/games

    alt cache vazio (primeira chamada)
        Catalog->>Redis: busca catalog:games:all
        Redis-->>Catalog: MISS
        Catalog->>SQL: SELECT jogos ativos
        SQL-->>Catalog: lista de jogos
        Catalog->>Mongo: agregacao das notas<br/>$match + $group
        Mongo-->>Catalog: media e contagem
        Catalog->>Redis: grava resultado (TTL 5 min)
        Catalog-->>Cliente: 200 OK — 716 ms
    else cache quente
        Catalog->>Redis: busca catalog:games:all
        Redis-->>Catalog: HIT
        Catalog-->>Cliente: 200 OK — 9,7 ms
    end

    Note over Catalog,Redis: qualquer escrita no catalogo<br/>invalida a chave
```

---

## 6. Segurança no gateway

Contraste entre requisição bloqueada e autorizada.

```mermaid
flowchart TB
    R1["Requisicao<br/>SEM token"]
    R2["Requisicao<br/>COM token valido"]

    K["Kong<br/>plugin JWT"]

    B["401 Unauthorized<br/>Server: kong/3.7.1"]
    S["Microsservico<br/>Server: Kestrel"]
    OK["200 OK"]

    R1 --> K
    R2 --> K
    K -->|"assinatura invalida<br/>ou ausente"| B
    K -->|"assinatura valida"| S
    S --> OK

    Nota["O trafego bloqueado<br/>nunca chega ao servico"]
    B -.-> Nota
```

---

## 7. Observabilidade

```mermaid
flowchart LR
    Users["UsersAPI<br/>/metrics"]
    Catalog["CatalogAPI<br/>/metrics"]

    Prom["Prometheus<br/>scrape a cada 10s"]
    Graf["Grafana"]

    P1["Latencia p95"]
    P2["Throughput"]
    P3["Status HTTP"]
    P4["Taxa de erro"]

    Users -->|"prometheus-net"| Prom
    Catalog -->|"prometheus-net"| Prom
    Prom --> Graf
    Graf --> P1
    Graf --> P2
    Graf --> P3
    Graf --> P4
```

---

## 8. Serverless: antes e depois

```mermaid
flowchart LR
    subgraph Antes["Antes — container 24/7"]
        A1["NotificationsAPI<br/>sempre de pe<br/>ocioso a maior parte<br/>do tempo"]
    end

    subgraph Depois["Depois — serverless"]
        D1["NotificationsFunction<br/>executa apenas quando<br/>chega mensagem"]
    end

    Fila{{"Fila do RabbitMQ"}}

    Antes -->|"migracao"| Depois
    Fila -->|"trigger"| D1
```

---

## Dicas de uso

- No Excalidraw: **More tools → Mermaid to Excalidraw**, cole o bloco e clique em *Insert*.
- O conversor suporta bem `flowchart` e `sequenceDiagram`. Se algum diagrama não converter, simplifique removendo os `subgraph`.
- Depois de inserido, tudo vira elemento editável: dá para reposicionar, mudar cores e destacar partes durante a apresentação.
- Sugestão de ordem na apresentação: diagrama 1 (contexto) → 2 (visão geral) → 3 e 4 (fluxos) → 5, 6, 7 e 8 (requisitos da fase, um a um).
