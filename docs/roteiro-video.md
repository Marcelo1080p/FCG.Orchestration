# Roteiro do vídeo — Tech Challenge Fase 3

Duração alvo: 15–18 min (limite de 20). Abertura com arquitetura, depois demonstração.

---

## Bloco 1 — Abertura (30s)

**Tela:** você na câmera.

> Olá, meu nome é Marcelo Cordeiro, sou aluno da Pós Tech da FIAP e este é o Tech Challenge da Fase 3 do projeto FIAP Cloud Games.
>
> Antes de mostrar a aplicação funcionando, quero explicar a arquitetura: o que a plataforma já era, o que mudou nesta fase e por quê. Assim, quando a gente for para a demonstração, cada peça já vai fazer sentido.

---

## Bloco 2 — Arquitetura (4 a 5 min)

### 2.1 De onde viemos — diagrama 1 (40s)

**Tela:** diagrama "Evolução do projeto" no Excalidraw.

> O projeto nasceu na Fase 1 como um monolito: uma API única, um banco, tudo no mesmo processo.
>
> Na Fase 2 ele foi decomposto em quatro microsserviços independentes — usuários, catálogo, pagamentos e notificações —, cada um com seu próprio repositório, seu próprio banco e comunicação assíncrona por mensageria.
>
> A Fase 3 não acrescenta funcionalidade de negócio: ela profissionaliza essa arquitetura. O problema que a gente tinha era concreto — cada microsserviço exposto direto na internet, nenhuma visibilidade sobre o que acontecia em produção, um serviço ocioso consumindo recurso 24 horas por dia, e um banco relacional sendo usado para tudo, inclusive para o que ele não é bom.

### 2.2 A arquitetura hoje — diagrama 2 (2 min)

**Tela:** diagrama "Visão geral da arquitetura". Vá apontando conforme fala.

> Esta é a arquitetura atual. Vou percorrer de fora para dentro.
>
> **Entrada.** Todo tráfego externo entra por um ponto único: o **Kong**, na porta 8000. Ele valida o token JWT e decide para onde encaminhar. Os microsserviços não ficam mais expostos individualmente — o cliente conhece um endereço só, e a política de segurança fica centralizada na borda.
>
> **Os serviços.** O **UsersAPI** cuida de cadastro e autenticação, e é quem emite os tokens. O **CatalogAPI** tem o catálogo de jogos, os descontos, as aquisições e as avaliações. O **PaymentsAPI** processa os pagamentos. Todos em .NET 8, com Clean Architecture e CQRS.
>
> **A comunicação.** Repare que não existe seta de um serviço para o outro. Eles não se chamam por HTTP — conversam por eventos, através do **RabbitMQ**. Quando um usuário se cadastra, o UsersAPI publica um `UserCreatedEvent` e segue a vida; ele não sabe nem se importa com quem vai consumir. Isso desacopla os serviços: se um cair, os outros continuam, e a mensagem espera na fila.
>
> **As notificações.** Aqui está a primeira novidade da fase. O que era um microsserviço rodando o tempo todo virou uma **função serverless**, acionada diretamente pela fila. Ela só executa quando chega mensagem.
>
> **A persistência.** Cada serviço tem seu próprio banco — nenhum acessa a tabela do outro. E o catálogo usa três tecnologias ao mesmo tempo: **SQL Server** para os jogos, **MongoDB** para as avaliações e **Redis** como cache. Já explico o porquê.
>
> **A observabilidade.** Por baixo de tudo, o **Prometheus** coleta métricas dos serviços e o **Grafana** apresenta em dashboard. É como a gente enxerga latência, volume e erro em tempo real.

### 2.3 As cinco decisões da fase (1min30)

**Tela:** pode ficar no diagrama 2, ou alternar para os diagramas 5 a 8 conforme cita.

> Resumindo as cinco entregas desta fase e o raciocínio por trás de cada uma:
>
> **Um — API Gateway.** Kong em modo declarativo, sem banco próprio: a configuração inteira é um arquivo YAML versionado no repositório. Ele valida o JWT antes de encaminhar, então requisição sem token nem chega ao serviço.
>
> **Dois — Serverless.** O serviço de notificações passava a maior parte do tempo parado, esperando evento. Manter container de pé para isso é desperdício. Virou Azure Function acionada por fila, com a infraestrutura descrita em Bicep, em plano de consumo — só custa quando executa.
>
> **Três — Observabilidade.** Escolhi a stack aberta, Prometheus e Grafana, em vez de uma plataforma paga. Motivo: não depende de conta externa e sobe junto com a aplicação por manifesto do Kubernetes — o dashboard inclusive é provisionado como código.
>
> **Quatro — NoSQL.** Implementei avaliações de jogos em MongoDB. É o caso certo para NoSQL: documento flexível, com lista de tags que no relacional exigiria tabela de junção, muita escrita e nenhuma necessidade de transação com o resto do catálogo.
>
> **Cinco — Cache.** O catálogo é a consulta mais frequente e muda pouco. Coloquei Redis na frente, com invalidação automática sempre que algo no catálogo muda. Já mostro o ganho medido.

### 2.4 Ponte para a demonstração (20s)

> Essa é a arquitetura. Agora vou subir tudo e mostrar funcionando de verdade: a plataforma inteira em containers, o gateway protegendo as rotas, a função serverless sendo acionada, o dashboard com métricas reais e o NoSQL integrado ao catálogo.

---

## Bloco 3 — Demonstração (10 a 12 min)

### 3.1 Repositórios (1 min)
**Tela:** GitHub.
- Mostre os 5 repositórios
- Abra o README do Orchestration: é o guia central
- Cite que cada microsserviço tem README próprio com endpoints e variáveis

### 3.2 Subindo com docker-compose (1min30)
**Tela:** terminal no FCG.Orchestration.

```bash
docker compose up -d --build
docker compose ps
```

> Um comando sobe a plataforma inteira: os três microsserviços, a função serverless, RabbitMQ, SQL Server, MongoDB, Redis, Kong, Prometheus e Grafana.

Mostre os 12 containers de pé.

### 3.3 Gateway: roteamento e segurança (2 min)
**Tela:** terminal.

Primeiro o bloqueio:
```bash
curl -i http://localhost:8000/api/users
```
> 401. E repare no cabeçalho: `Server: kong`. Quem barrou foi o gateway — a requisição não chegou ao microsserviço.

Depois o caminho autorizado:
```bash
curl -X POST http://localhost:8000/api/auth/login -H "Content-Type: application/json" -d "{\"email\":\"admin@fcg.com\",\"password\":\"Admin@123\"}"
```
> Token emitido pelo UsersAPI, através do gateway.

```bash
curl http://localhost:8000/api/users -H "Authorization: Bearer <token>"
```
> Agora passou. Mesma rota, mesma porta — a diferença é o token válido.

### 3.4 Fluxo de cadastro e a função serverless (2 min)
**Tela:** terminal dividido — um lado os comandos, outro `docker compose logs -f notificationsfunction`.

- Registre um usuário pelo gateway
- Mostre a função sendo acionada nos logs: `Executing 'Functions.UserCreatedFunction' (Reason='RabbitMQ message detected...')`

> Repare: ninguém chamou essa função. Ela foi acionada pela mensagem na fila.

**Momento forte (opcional, +1min):** derrube a função, registre outro usuário, mostre a mensagem acumulada na fila no painel do RabbitMQ, suba a função e mostre ela consumindo.

> Nenhum evento se perde.

### 3.5 Fluxo de compra completo (2 min)
**Tela:** terminal + logs.

- Login como admin, cadastra um jogo
- Login como usuário comum, adquire o jogo
- Mostre a cadeia: CatalogAPI publica → PaymentsAPI aprova → função notifica

> Três serviços participaram e nenhum chamou o outro diretamente.

### 3.6 NoSQL e cache no catálogo (2 min)
**Tela:** terminal.

- Avalie o jogo: `POST /api/games/{id}/reviews` com nota e tags
- Liste o catálogo e mostre `averageRating` e `reviewCount` aparecendo

> A nota média não é calculada em memória: é uma agregação do próprio MongoDB, uma consulta só para o catálogo inteiro.

- Mostre o documento no Mongo:
```bash
docker exec fcg-mongo mongosh FCG_CatalogDB --quiet --eval "db.reviews.find().toArray()"
```
> Repare no array de tags — no relacional isso seria uma tabela de junção.

- Cache: chame `/api/games` duas vezes cronometrando
```bash
curl -s -o NUL -w "%{time_total}s\n" http://localhost:8000/api/games
```
> Primeira chamada vai ao banco. Segunda vem do Redis. De 716 milissegundos para 9,7 — cerca de 70 vezes mais rápido.

### 3.7 Observabilidade (2 min)
**Tela:** Grafana em `localhost:3000`.

- Mostre o dashboard com dados reais
- Aponte: latência p95, throughput, status codes, taxa de erro
- Gere tráfego ao vivo e mostre os gráficos reagindo

> Um detalhe interessante: o UsersAPI tem latência bem maior que o CatalogAPI. Faz sentido — o login roda BCrypt, que é caro de propósito, enquanto o catálogo responde do cache.

### 3.8 Kubernetes (1min30)
**Tela:** terminal.

```bash
kubectl apply -f k8s/
kubectl get pods
```
> Todos os manifests estão consolidados neste repositório: Deployment, Service, ConfigMap e Secret de cada peça.

Mostre os pods rodando.

---

## Bloco 4 — Encerramento (30s)

> Recapitulando: gateway como entrada única com segurança na borda, notificações em serverless acionadas por fila, observabilidade com Prometheus e Grafana, persistência poliglota com MongoDB e cache em Redis — tudo containerizado e com manifestos para Kubernetes.
>
> Os links dos repositórios estão no relatório de entrega. Obrigado.

---

## Checklist antes de gravar

- [ ] `docker compose up -d --build` rodando e os 12 containers de pé
- [ ] Um jogo já cadastrado (para não gastar tempo criando na hora)
- [ ] Gerar tráfego alguns minutos antes, para o Grafana ter histórico nos gráficos
- [ ] Kubernetes habilitado no Docker Desktop, se for demonstrar
- [ ] Token de admin já obtido e copiado, para agilizar
- [ ] Diagramas abertos no Excalidraw, em abas separadas
- [ ] Terminal com fonte grande (legibilidade no vídeo)
