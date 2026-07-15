# GeoDB Cities — Clustering K-Means de Cidades

Aplicação web para explorar cidades do mundo via [GeoDB Cities API](https://rapidapi.com/wirefreethought/api/geodb-cities) (RapidAPI), coletar grandes volumes de dados em paralelo usando Web Workers e aplicar **clustering K-Means** (por latitude, longitude e população) diretamente no navegador — com suporte a `SharedArrayBuffer` para troca eficiente de dados entre threads.

O backend em **Node.js/Express** atua como proxy autenticado para a API do GeoDB (evitando expor a chave da RapidAPI no cliente) e serve o dashboard em EJS. Um container **Nginx** fica na frente como reverse proxy.

## Funcionalidades

- **Exploração paginada** de cidades (nome, país, população, coordenadas) via API GeoDB.
- **Seleção manual** de cidades de interesse para acompanhar na análise.
- **Coleta paralela** de milhares de cidades usando múltiplos Web Workers, com retry/backoff em caso de rate limit (HTTP 429) e tratamento de limite de plano (HTTP 403).
- **Cache em duas camadas**: arquivo estático (`sample-cities.json.bak`) → IndexedDB (com fallback em `localStorage`) → rede, para evitar recoletar dados repetidamente.
- **K-Means** configurável (número de clusters `k`), executado em Web Workers, com normalização de features e memória compartilhada (`SharedArrayBuffer`) quando o navegador permite (`crossOriginIsolated`).
- **Dashboard** com KPIs, barra de progresso da coleta e visualização dos clusters em gráfico (Chart.js).
- **Healthcheck** HTTP (`/api/health`) usado pelo Docker Compose para orquestrar a subida dos serviços.

## Arquitetura

```
├── app/                      # Aplicação Node.js/Express
│   ├── indice.js             # Entry point do servidor
│   ├── controllers/          # Controllers HTTP (views e API)
│   ├── routes/                # webRoutes (views) e apiRoutes (JSON)
│   ├── services/              # Integração com a GeoDB API (axios)
│   ├── middleware/            # Validação de query params e error handler
│   ├── util/                  # Validação de ambiente, query string builder
│   ├── views/                  # Templates EJS (dashboard)
│   └── public/js/
│       ├── controllers/       # Orquestração da UI (paginação, seleção, coleta, clustering)
│       ├── models/            # Estado da aplicação (paginação, cidades selecionadas)
│       ├── views/              # Renderização de listas de cidades e clusters
│       ├── services/           # Lógica de coleta paralela e K-Means
│       ├── workers/            # Web Workers de coleta e K-Means
│       └── util/               # Cache (IndexedDB) e serialização para SharedArrayBuffer
├── nginx/                     # Reverse proxy (Nginx) na frente do Node
├── docker-compose.yml         # Orquestra os serviços node + nginx
└── Dockerfile                 # Build da aplicação Node
```

**Fluxo de dados:** navegador → Nginx (porta 80) → Node/Express (porta 3000) → GeoDB API (RapidAPI). A chave da RapidAPI (`RAPIDAPI_KEY`) nunca é exposta ao cliente: todas as chamadas passam pelo endpoint interno `/api/cities`.

## Stack técnica

| Camada | Tecnologias |
|---|---|
| Backend | Node.js, Express 5, EJS, Axios, dotenv |
| Frontend | JavaScript ES Modules (vanilla), Bootstrap 5, Chart.js, Web Workers, IndexedDB, SharedArrayBuffer |
| Infra | Docker, Docker Compose, Nginx |

## Pré-requisitos

- Docker e Docker Compose **ou** Node.js 20+ (para rodar localmente sem containers)
- Uma chave de API válida da [GeoDB Cities API na RapidAPI](https://rapidapi.com/wirefreethought/api/geodb-cities)

## Configuração

Crie um arquivo `.env` na raiz de `geodb-cities/` com:

```env
PORT=3000
RAPIDAPI_KEY=sua_chave_aqui
```

> ⚠️ **Atenção:** o `.env` presente neste repositório contém uma chave de API real exposta. Ela deve ser **revogada/rotacionada imediatamente** no painel da RapidAPI e o arquivo `.env` não deve ser commitado (adicione-o ao `.gitignore`, que atualmente não o exclui).

## Como rodar

### Com Docker (recomendado)

```bash
cd geodb-cities
docker compose up --build
```

- Aplicação (via Nginx): http://localhost
- Aplicação (direto no Node): http://localhost:3000/geo/cities
- Healthcheck: http://localhost:3000/api/health

O `docker-compose.yml` monta o código-fonte como volume e usa `nodemon`, então alterações em `app/` reiniciam o servidor automaticamente.

### Localmente (sem Docker)

```bash
cd geodb-cities/app
npm install
npm run dev    # com nodemon (hot-reload)
# ou
npm start       # produção
```

Acesse: http://localhost:3000/geo/cities

## Endpoints da API interna

| Método | Rota | Descrição |
|---|---|---|
| GET | `/geo/cities` | Renderiza o dashboard (EJS) |
| GET | `/api/cities` | Proxy para a GeoDB API (`limit`, `offset`, `sort`, `location`, `radius`, `countryIds`, `namePrefix`, etc.) |
| GET | `/api/health` | Status de saúde do serviço (usado pelo Docker healthcheck) |

O `limit` é validado entre 1 e 10 (limite do plano gratuito da RapidAPI) e `sort` aceita apenas `name`, `population`, `elevationMeters` ou `timezone`.

## Notas sobre o K-Means

- Os pontos são normalizados (min-max) por latitude, longitude e população antes do cálculo de distância euclidiana.
- A atribuição de pontos a centróides é distribuída entre Web Workers; quando o contexto de navegação está `crossOriginIsolated` e `SharedArrayBuffer` está disponível, os dados são trocados via memória compartilhada em vez de `postMessage`, reduzindo overhead de serialização.
- Sem esse isolamento (a maioria dos ambientes de desenvolvimento sem headers COOP/COEP configurados corretamente), a aplicação recorre automaticamente a `postMessage` como fallback — o dashboard sinaliza qual modo está ativo.

## Branches

- `main`: branch principal
- `geoCitiesV2`: branch de desenvolvimento da versão atual (dashboard de análise, clustering, coleta paralela)
