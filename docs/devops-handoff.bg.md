# Предаване към DevOps — Инфраструктура и разгръщане на платформата

Кратко описание на една страница за DevOps екипа: какво пускаме, хранилищата за данни, как
се build-ват и разгръщат контейнерите и препоръчителни параметри на сървърите. За пълни
детайли вижте [11 — Deployment & Operations](./11-deployment.md),
[10 — Phase-1 Co-Deployment](./10-phase1-co-deployment.md) и
[08 — Database Architecture](./08-database-architecture.md).

---

## 1. Накратко

- **Едно Git repo → един Docker image.** Всеки service е същият image; env променливата
  `SERVICES` определя кои service-и пуска даден контейнер.
- **13 логически service-а**, пускани в production като **7 приложни deployable-а** (всеки с
  `web` + `worker` контейнер) + **gateway** като единствена публична входна точка.
- **База-данни-на-service**: 12 логически Postgres бази в **един PostgreSQL 16 клъстер**
  (отделна ограничена роля за всяка база, без cross-schema join-ове) + **отделен TimescaleDB
  клъстер** за IoT.
- Споделени хранилища: **Redis 7**, **Qdrant** (вектори), **S3 / MinIO** (файлове),
  **TimescaleDB** (IoT времеви редове).
- Плюс няколко **self-hosted външни engine-а** (LiteLLM, Carbone, Unstructured, Nango,
  Documenso, Authentik, Node-RED) във вътрешната мрежа.
- Цел за Phase-1: **един достатъчно оразмерен хост** (Docker Compose). Kubernetes **не е**
  необходим за коректност — мащабирайте по-късно чрез добавяне на реплики.

---

## 2. Service-и (кратки описания)

Всички service-и са FastAPI (Python 3.12) приложения. Портовете по-долу са dev портовете в
контейнера; в production се достигат през вътрешната мрежа.

| Service | Порт | Какво прави |
|---|---|---|
| **gateway** | 8000 | Единствената публична входна точка. Reverse proxy, JWT/JWKS валидиране, rate limiting, CORS, препредаване на webhook-ове. **Няма база данни.** |
| **identity-service** | 8010 | Потребители, наематели (фирми), вход (Authentik OIDC + локален), издаване на JWT, RBAC роли/права. |
| **agent-service** | 8020 | LangGraph AI агентен runtime + история на разговорите (сесии/съобщения). Горещият streaming път. |
| **model-gateway** | 8030 | Единственият service, който говори с LLM доставчиците (през LiteLLM). Отчитане на токени, kill switch. |
| **knowledge-service** | 8040 | Библиотека с документи, парсване/OCR, embeddings + векторно търсене (RAG), WebDAV/Drive синхронизация. |
| **registry-service** | 8050 | Динамични потребителски таблици (CRM, договори, задачи и др.), шаблони, одит. |
| **business-service** | 8100 | Типизиран ERP домейн: фактуриране, складова наличност/счетоводна книга, разходи. |
| **document-service** | 8060 | Шаблони за документи + рендиране (PDF/DOCX/XLSX през Carbone), ценови листи, надценки, КСС. |
| **billing-service** | 8070 | Баланси с токени, отчитане на потребление, Stripe checkout + webhook-ове, пакети. |
| **integration-service** | 8080 | Външна свързаност: Google/Gmail, IMAP/SMTP, WebDAV, електронен подпис (Documenso), OAuth (Nango). |
| **platform-service** | 8090 | Известия (email/Telegram/в приложението), тикети за поддръжка, централен одит sink, настройки. |
| **iot-service** | 8110 | Регистър на IoT устройства/сензори, съхранение на времеви редове (TimescaleDB), continuous aggregates, известяване по наемател. Показанията пристигат нормализирани от Node-RED ingestion edge. |
| **manufacturing-service** | 8120 | Производствени поръчки, BOM/маршрути, MRP, планиране на капацитет, MES терминали + SCADA↔ERP обмен. |

---

## 3. Инфраструктура и хранилища за данни

### 3.1 Бази данни (PostgreSQL 16, един клъстер)

База-данни-на-service: **12 логически бази в един PostgreSQL 16 клъстер** (всяка с отделна
login роля, която вижда само своята база — без cross-schema join-ове, наложено чрез grant-ове)
+ **един отделен TimescaleDB клъстер** за `iot` базата с времеви редове.

| База | Service-собственик | Особености |
|---|---|---|
| `identity` | identity-service | RS256 ключова двойка за JWKS |
| `agent` | agent-service | LangGraph checkpoint таблици |
| `modelgw` | model-gateway | AES-GCM криптирани креденшъли на доставчици |
| `knowledge` | knowledge-service | работи в комбинация с **Qdrant** за вектори |
| `registry` | registry-service | JSONB редове, Row-Level Security |
| `document` | document-service | JSONB шаблони |
| `billing` | billing-service | само-добавяема (append-only) книга за потребление |
| `integration` | integration-service | AES-GCM трезор за креденшъли (не-OAuth) |
| `platform` | platform-service | известия, одит, поддръжка |
| `business` | business-service | последователности за фактури по наемател |
| `iot` | iot-service | **отделен TimescaleDB клъстер** (Postgres 16 + Timescale extension): hypertables, continuous aggregates, retention/компресия |
| `manufacturing` | manufacturing-service | последователности за поръчки по наемател, RLS |

> `iot` работи на собствен **TimescaleDB** клъстер (Postgres 16 + Timescale extension).
> Останалите 12 са логически бази в споделения клъстер — принципът база-данни-на-service се
> запазва.

### 3.2 Споделени хранилища

| Компонент | Image | За какво |
|---|---|---|
| **Redis 7** | `redis:7` (с AOF) | Кеш, rate limits, `arq` опашки за задачи, event bus (Redis Streams). В production с AOF + реплика. |
| **Qdrant** | `qdrant/qdrant` | Векторно хранилище за RAG на knowledge-service. |
| **S3 / MinIO** | `minio/minio` (dev) / произволен S3 в prod | Оригинали на документи, генерирани PDF/XLSX/DOCX, прикачени файлове от чата. Отделен bucket за всеки service-собственик. |

### 3.3 Self-hosted външни engine-и (само вътрешна мрежа)

Всеки стои зад порт, притежаван от точно един service. Никой не е публично достъпен за
приложен трафик.

| Engine | Image | Собственик | Endpoint | Stateful? |
|---|---|---|---|---|
| **LiteLLM** | `ghcr.io/berriai/litellm` | model-gateway | `litellm:4000` | Не |
| **Carbone** | build (`infra/carbone`) | document-service | `carbone:4000` | Не |
| **Unstructured** | `unstructured-io/unstructured-api` | knowledge-service | `unstructured:8000` | Не |
| **Nango** | `nangohq/nango-server` | integration-service | `nango:3003` / `:3009` | **Да** (собствен pg/redis) |
| **Documenso** | `documenso/documenso` | integration-service | `documenso:3000` | **Да** (собствен pg) |
| **Authentik** | self-hosted | identity (OIDC IdP) | вътрешен | **Да** (собствен pg) |
| **Node-RED** | `nodered/node-red` | iot-service (ingestion edge) | `node-red:1880` | **Да** (собствен volume) |

> Node-RED е IoT ingestion edge: декодира индустриални протоколи (MQTT/Modbus/OPC UA) и
> публикува нормализирани показания към iot-service **през gateway-а** (не е обвит зад порт).
> Editor UI-ят е само за служители, зад Authentik SSO — никога публичен.
>
> Опционално: **Ollama** (локален LLM, зад LiteLLM) за цена/поверителност и **n8n** (вътрешен
> automation plane). Пускайте ги само при нужда.

---

## 4. Как се build-ват и разгръщат контейнерите

### 4.1 Един image, много роли

- Всички service-и се build-ват от **един `Dockerfile`** в корена на repo-то (`python:3.12-slim`,
  `uv` инсталира целия workspace). CI build-ва и кешира този image **веднъж**, тагнат на commit.
- Поведението на контейнера се задава изцяло от **env променливата `SERVICES`**, напр.
  `SERVICES=agent,model-gateway,knowledge`. Entrypoint-ът mount-ва само тези router-и; worker
  entrypoint-ът регистрира само тези фонови задачи.
- **`web` и `worker` са отделни контейнери** от един и същ image, за всеки deployable — така
  тежка фонова задача (embedding sweep, нощен MRP) никога не задушава HTTP латентността.

### 4.2 Production карта на deployable-ите (7 + gateway)

| Deployable | Стойност на `SERVICES` | Worker? | Публичен? |
|---|---|---|---|
| **gateway** | `gateway` | не | **Да — единственият** |
| **identity** | `identity` | да | не |
| **ai-plane** | `agent,model-gateway,knowledge` | да | не |
| **erp-core** | `registry,business,document` | да | не |
| **ops-plane** | `billing,integration,platform` | да | не |
| **iot** | `iot` | да | не |
| **manufacturing** | `manufacturing` | да | не |

> `iot` и `manufacturing` работят **разделени от първия ден** (собствени deployable-и) —
> техните профили на натоварване (ingest на времеви редове, нощни MRP пускания) нямат нищо
> общо с основната крива на мащабиране, а `iot` се нуждае от отделния TimescaleDB клъстер.

Групирането е по **профил на мащабиране + граница на доверие + statefulness**, не по домейн.
Отделянето на която и да е група в собствен контейнер по-късно е **промяна в конфигурацията**
(нов контейнер, насочване на `DATABASE_URL`, превключване на `deps.py` адаптер, обновяване на
routing таблицата на gateway-а) — никога пренаписване.

### 4.3 Процес на разгръщане

1. **Build** на единия image, таг на commit.
2. **Миграции**: задача `migrate` (същият image) пуска `alembic upgrade head` за всеки service
   в неговия deployable **преди** новият код да поеме трафик. Миграциите са обратно съвместими
   (expand/contract), така че rolling разгръщанията са без downtime.
3. **Rolling** на `web` / `worker` контейнерите. Gateway-ът пропуска трафик при `/ready`.
4. Мрежа: **TLS завършва на gateway-а** (или reverse proxy пред него). Всичко останало е само
   вътрешно. Webhook-овете (Stripe, Brevo, Documenso, Nango) минават през gateway-а със запазено
   raw тяло.

### 4.4 Конфигурация и тайни (secrets)

- 12-factor: **цялата конфигурация през env променливи**. Dev използва git-игнориран `.env`;
  prod инжектира от **secret manager** при разгръщане.
- **Тайните никога в код или image-и.** Един собственик за всеки външен креденшъл (LLM ключове →
  model-gateway, Stripe → billing, Nango/Documenso → integration, Brevo/Telegram → platform).
- Всеки service се свързва със **собствена DB роля** (собствен `DATABASE_URL`).

### 4.5 Health и наблюдаемост

- `GET /health` (liveness) и `GET /ready` (DB + Redis достъпни — оркестраторът пропуска трафик
  на база това).
- **Структурирани JSON логове** (structlog) с `request_id`, `user_id`, `company_id`, `service`.
- **OpenTelemetry** trace-ове (gateway-ът стартира trace-а, всеки hop го продължава) + **Sentry**
  за грешки.

---

## 5. Препоръки за сървъри

> Това са начални препоръки за разгръщане на един хост (phase-1), обслужващо SME наематели.
> Настройвайте спрямо реалното натоварване — архитектурата мащабира чрез добавяне на реплики,
> не чрез преархитектуриране.

### 5.1 Phase-1 — един приложен хост

Пуска gateway + 7 приложни deployable-а (всеки с web + worker) и stateless backing engine-ите
(LiteLLM, Carbone, Unstructured, Node-RED). Облачните LLM-и (Anthropic) означават, че GPU не е
необходим.

| Ресурс | Препоръка |
|---|---|
| **vCPU** | 8–12 vCPU (минимум 4) — iot ingest-ът + нощните MRP worker-и добавят натоварване върху основата |
| **RAM** | 24–32 GB (горещият LLM/RAG път и рендирането на документи са водещите по памет) |
| **Диск** | 100+ GB SSD за хоста; обектното хранилище отделно (виж по-долу) |
| **ОС** | Ubuntu LTS (или произволен модерен Linux с Docker Engine + Compose v2) |

### 5.2 Слой за данни (препоръчват се отделен хост / managed инстанции)

| Компонент | Препоръка |
|---|---|
| **PostgreSQL 16** | Силно препоръчителна managed инстанция. Започнете с 4 vCPU / 16 GB RAM / 100 GB SSD с място за растеж; включете автоматични backup-и + PITR. |
| **Redis 7** | 1–2 vCPU / 4 GB RAM, включена AOF персистентност, + реплика в prod. |
| **Qdrant** | 2 vCPU / 4–8 GB RAM, SSD volume (расте с броя документи/вектори). |
| **S3 / MinIO** | S3 в prod (без оразмеряване) или MinIO хост с достатъчно диск + включено versioning. |
| **TimescaleDB** (IoT) | Отделен клъстер (Postgres 16 + Timescale extension). Започнете с 4 vCPU / 16 GB RAM / SSD; оразмерете диска спрямо обема ingest на времеви редове + прозореца за retention. Политиките за компресия/retention ограничават растежа на суровите редове. |

### 5.3 Път за хоризонтално мащабиране (по ред, когато се появи натоварване)

1. **Реплики на gateway** зад load balancer (stateless — тривиално).
2. **Реплики на ai-plane** — горещият chat/LLM път; мащабирайте web и worker независимо.
3. **Отделяне на най-натоварения член** (model-gateway или knowledge) при поява на конкуренция.
4. **Отделяне на членове на erp-core / ops-plane** според индивидуалното натоварване или
   собствеността от екип.

Всяка стъпка е промяна в конфигурацията/топологията (брой реплики, стойност на `SERVICES`,
routing таблица) върху **същия image**.

### 5.4 Бележка за GPU

Не е необходим за базовата настройка (LLM-ите са облачни Anthropic през LiteLLM). GPU хост е
нужен само ако self-host-вате локални модели (Ollama) за цена/поверителност — отделете го на
собствен node с GPU.

---

## 6. Backup-и и устойчивост

| Актив | Механизъм |
|---|---|
| PostgreSQL (всички бази) | Планирани `pg_dump` / PITR; backup по отделна база, за да може възстановяване да е по service. |
| Redis | AOF + реплика; опашаните събития/задачи преживяват рестарт. |
| Обектно хранилище | S3/MinIO versioning + lifecycle политики. |
| Критични billing събития | Транзакционен outbox при продуцента — загуба на събитие изисква загуба на базата, не само на Redis. |
| Външни backer-и | Nango (собствен pg/redis), Documenso (собствен pg), Authentik (собствен pg) — backup-вайте и тях. |

---

## 7. Бърза справка (dev)

| Задача | Команда |
|---|---|
| Стартиране на всичко | `docker compose up --build` |
| Пускане на миграции | `docker compose run --rm migrate` |
| Следене на един deployable | `docker compose logs -f ai-plane` |
| Мащабиране на gateway | `docker compose up -d --scale gateway=3` |

Вижте [10 §7](./10-phase1-co-deployment.md#7-sample-dev-docker-compose-sketch) за референтния
`docker-compose.yml`.
