# 06 — Обяснение на архитектурните модели

Справочник за всеки именуван pattern, върху който стъпва архитектурата: какво представлява
pattern-ът, къде се появява в тази система, защо е избран пред алтернативите и каква цена има.
Прочетете това, ако термин от другите документи ("port", "strangler", "thin event", …) е
непознат или ако искате reasoning-а зад дадено design rule.

---

## 1. Структурни patterns (форма на системата)

### 1.1 Microservices с bounded contexts (DDD)

**Какво представлява.** Системата е разложена на независимо deploy-ваеми услуги, всяка
подравнена с *bounded context* от Domain-Driven Design — бизнес област със собствен
речник, модел и invariants (identity, billing, knowledge, registries, …).

**Къде.** 10-те услуги (11 с post-parity business-service) в
[02-service-catalog.md](./02-service-catalog.md). Границите не са измислени от нулата:
те следват module clusters с най-висока cohesion, които вече се виждат в монолита
(`core/auth`, `core/registries`, `core/payments`, …), което е най-безопасният начин
да се начертаят service lines — по граници, които domain-ът вече е доказал. Приложено е
и обратното правило: граници, които не оправдават собствен deployable, остават модули
(conversation history в agent-service; notifications/support/audit/settings в един
platform-service — виж
[02 § Deliberately merged](./02-service-catalog.md#deliberately-merged-boundaries)).

**Защо.** Независимо мащабиране на hot paths (chat срещу CRUD срещу PDF rendering),
failure isolation и възможност agent platform да се развива агресивно, без да destabilize-ва
billing или auth.

**Компромис.** Distributed-systems complexity: network failures между услуги, повече
deployment units, cross-service debugging. Затова архитектурата комбинира pattern-а с две
mitigations: строга call-matrix (само agent-service fan-out-ва) и правилото
*"boundaries are code boundaries first, deployment boundaries second"* — услугите могат да бъдат
co-deploy-вани в един container, докато load не наложи разделяне.

### 1.2 API Gateway

**Какво представлява.** Една edge услуга, през която влиза целият външен traffic. Тя притежава
cross-cutting concerns (authentication, rate limiting, routing, streaming passthrough) и
съдържа нула business logic.

**Къде.** Услугата `gateway` ([01 §5](./01-architecture-overview.md#5-the-api-gateway)).

**Защо.** Три причини, специфични за тази система: (a) множество текущи и бъдещи frontends
(web, mobile, Telegram/Viber) трябва да удрят идентични APIs с идентична security; (b) урокът
от монолита — неговото route-mount *ordering* в `server.js` беше критично, а rate limiter-ът му
беше дублиран 14 пъти — аргументира точно едно място за edge policy; (c) по време на
миграцията routing table на gateway-а *е* strangler механизмът (§4.1).

**Компромис.** Gateway е single point of failure и потенциален bottleneck — трябва да остане
thin (без aggregation, без transformation) и да е първото нещо, което се replicate-ва.

### 1.3 Database-per-Service

**Какво представлява.** Всяка услуга притежава своето data store изключително. Никоя друга
услуга не може да се свързва към него; cross-service data минава през HTTP APIs или events.

**Къде.** Всяка DB-owning услуга има собствена logical PostgreSQL база
(`identity`, `registry`, `knowledge`, …). Vector store-ът (pgvector) принадлежи само на
knowledge-service; agents го query-ват единствено през `/search` API.

**Защо.** Единната schema на монолита с ~80 `core.*` таблици позволяваше всеки модул да
join-ва всяка таблица — invisible coupling, което превръщаше всяка промяна в platform-wide
risk. Private databases правят всяка dependency explicit, versionable contract и позволяват
на всяка услуга да избере правилната storage shape (JSONB rows за registries, vectors за knowledge).

**Компромис.** Няма cross-service joins и няма cross-service transactions. Aggregations се
местят в owning service (dashboard briefing живее в registry-service); consistency между
услуги става *eventual* и се обработва с idempotent event consumers (§3.2).

### 1.4 Multi-tenancy с defense-in-depth (RLS)

**Какво представлява.** Всички tenants споделят едни и същи услуги и бази; всеки row носи
tenant ID, всяка query е tenant-scoped, а PostgreSQL Row-Level Security налага scope-а втори
път на database layer.

**Къде.** Tenant context (`X-Company-Id`) се проверява в gateway-а, propagate-ва се в
headers и се прилага във всяка услуга. RLS policies mirror-ват съществуващия GUC подход
на монолита (`app.current_tenant`).

**Защо.** Tenant isolation bugs са най-вредният клас bug в B2B платформа. Два независими
слоя (application scoping + RLS) означават, че забравен `WHERE company_id = …` става query,
която не връща нищо, не data leak.

**Компромис.** RLS добавя friction към migrations и debugging (sessions трябва да set-ват GUC),
и леко усложнява admin cross-tenant reads (explicit bypass role, audited).

---

## 2. Patterns вътре във всяка услуга

### 2.1 Hexagonal architecture (Ports & Adapters)

**Какво представлява.** Domain core зависи само от **ports** — interfaces (Python
`Protocol`s), описващи от какво има нужда domain-ът (`LLM`, `Retriever`, `ConversationStore`,
`VectorStore`). **Adapters** implement-ват тези ports и са единствените модули, на които е
позволено да import-ват driver, SDK или да знаят URL. Dependencies винаги сочат навътре:
edge → domain ← adapters.

**Къде.** Стандартният service layout в [02](./02-service-catalog.md) (`routes/ →
services/ → ports/ ← adapters/`) и swap recipes в
[03 §5](./03-agent-platform.md#5-swapping-infrastructure).

**Защо.** Прави скъпите неща евтини: смяна на vector store, пренасочване на tool adapter
от монолита към нова услуга по време на миграцията или замяна на LLM provider е *един нов
adapter + един wiring line* — domain-ът и всеки agent остават untouched. Освен това прави
domain logic тестируема с in-memory fakes вместо mocked HTTP.

**Компромис.** Повече файлове и ниво на indirection, което изглежда като ceremony в малки
услуги. Дисциплината се отплаща само ако се enforce-ва — затова driver import извън
`adapters/` fail-ва code review.

### 2.2 Repository pattern + DTO/model separation

**Какво представлява.** Database access е encapsulated в repository classes зад ports, а
shapes, които пресичат API-то (`Pydantic` DTOs), са умишлено различни от storage shapes.

**Къде.** `adapters/` layer-ът на всяка DB-owning услуга; правилото "DTOs ≠ DB models" в
[03 §5](./03-agent-platform.md#case-b--swap-a-database-in-a-db-owning-service).

**Защо.** API contract може да остане стабилен, докато storage се развива (добавяне на
columns, промяна на indexes, дори смяна на engines). Монолитът вече използваше repositories
(35 от тях) — това пренася навика и формализира DTO boundary, която му липсваше.

**Компромис.** Mapping code. Държи се поносимо чрез `model_validate` на Pydantic и като не
се измислят DTOs там, където storage shape наистина е contract-ът.

### 2.3 Dependency injection с composition root

**Какво представлява.** Objects не създават собствените си dependencies; всичко се wiring-ва
на едно място — `deps.py` — чрез dependency system на FastAPI. Този файл е *composition root*:
единственото място, където adapters се instantiate-ват и bind-ват към ports.

**Къде.** Всяка услуга; всички "swap infrastructure" recipes завършват с "rewire one line of
`deps.py`".

**Защо.** Swappability (§2.1) е реална само ако има точно една wiring point. Освен това
дава на tests една ясна точка за override (inject fakes), без monkeypatching.

**Компромис.** Indirection — за да намерите *кой* concrete class обслужва даден port, гледате
`deps.py`, не call site-а. Това е целта, но изненадва newcomers.

---

## 3. Communication patterns

### 3.1 Synchronous request/response с context propagation

**Какво представлява.** Service-to-service calls са обикновен HTTP (httpx, pooled,
internal network), които носят verified identity headers (`X-User-Id`, `X-Company-Id`,
`X-Roles`, `X-Request-Id`) плюс краткоживеещ **service token** — client-credentials style
доказателство, че caller-ът е platform service, не просто нещо в network-а.

**Къде.** Всички стрелки в call matrix ([02](./02-service-catalog.md)); правилата за header
injection в [01 §5](./01-architecture-overview.md#5-the-api-gateway).

**Защо.** Използва се за *queries and commands that need an answer now* (agent tools четат
registry rows). HTTP + OpenAPI държи contracts explicit и позволява TS client да се генерира.

**Компромис.** Latency на hop и temporal coupling (callee трябва да е up). Mitigate-ва се
чрез shallow call graph (agent-service е единственият fan-out hub) и чрез преместване на
всичко, което *не* изисква незабавен отговор, към events (§3.2).

### 3.2 Event-driven pub/sub с thin events и idempotent consumers

**Какво представлява.** Услугите publish-ват *facts* (`tenant.created`, `token.usage`,
`document.ingested`) към Redis Streams topics; consumer groups ги обработват независимо.
Events са **thin** — IDs и минимален payload; consumers fetch-ват details през HTTP, ако трябва.
Всеки consumer е **idempotent**, защото delivery е at-least-once.

**Къде.** [01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events) — topic
таблицата и fan-out диаграмата.

**Защо.** Decouple-ва producers от consumers: model-gateway не знае, че billing съществува;
registry-service seed-ва system registries при tenant creation, без identity-service да знае
за registries. Нови consumers се attach-ват без промяна в producers — същото open/closed
свойство, което има agent platform.

**Компромис.** Eventual consistency (token balance изостава от LLM call-а с milliseconds до
seconds) и оперативната нужда да се monitor-ват consumer lag и dead-letter handling. Когато
event *не трябва* да се изгуби спрямо DB write, producing service използва outbox table,
flush-вана от worker-а му (запис на row и pending event в една transaction). `token.usage`
е в този клас — billing data е, затова model-gateway винаги го publish-ва през outbox-а си;
Redis работи с AOF persistence + replica в production за всичко останало (виж
[01 §7](./01-architecture-overview.md#7-asynchronous-work-and-events)).

### 3.3 Job queue / private workers

**Какво представлява.** Всяка услуга притежава arq workers, които четат собствени Redis queues
за deferred или heavy work (embedding, file sync, email sends, retention purges). Queues са
private — никоя услуга не enqueue-ва в чужда.

**Къде.** `workers/` папката на всяка услуга; job lists по услуги в
[02](./02-service-catalog.md).

**Защо.** Държи request latency равна и изолира background load — монолитът научи това по
трудния начин, когато Drive sync изтощи general worker-а му и наложи `sync-worker.js` split.
Private queues пазят database-per-service принципа за work, не само за data.

**Компромис.** Job state е още нещо за наблюдение; cross-service workflows трябва да
комуникират през events, никога чрез reach into queue на съседна услуга.

---

## 4. Evolution & extensibility patterns

### 4.1 Strangler Fig (migration)

**Какво представлява.** Вместо big-bang rewrite, facade (gateway-ът) застава пред legacy
системата; нови услуги поемат route-by-route, докато старата система вече не обслужва нищо
и се премахва — новата система постепенно "strangle"-ва старата.

**Къде.** Целият migration plan в
[05 §5](./05-migration-pros-and-cons.md#5-risk-reducing-migration-strategy-strangler-not-big-bang):
gateway proxy-ва всичко към монолита на ден първи, после всяка фаза flip-ва path prefixes
към нови услуги. Agent tools първо използват adapter clients, насочени към монолита, а по-късно
към новите услуги — config flip благодарение на ports & adapters (§2.1).

**Защо.** Реални tenants използват платформата ежедневно; cutover, който може да се движи
(и връща назад) по route и по tenant, превръща existential risk в поредица от малки рискове.

**Компромис.** Двойна infrastructure и двойно bookkeeping за периода, плюс feature freeze
на монолита, за да остане parity фиксирана цел.

### 4.2 Plugin architecture: convention over configuration + registry discovery

**Какво представлява.** Разширяването на системата означава *добавяне на файлове по конвенция*,
не редакция на core code. Registry открива extensions при startup чрез filesystem scanning
(reflection), validate-ва manifest-ите им и ги mount-ва; счупено extension се skip-ва, никога
не е fatal.

**Къде.** Три места, умишлено със същата форма:
- **Agents** — `app/agents/<id>/manifest.yaml` + optional `graph.py`
  ([03 §1](./03-agent-platform.md#1-how-extensibility-is-achieved)).
- **Tools** — един module на tool + един registry list
  ([03 §4](./03-agent-platform.md#4-tools-the-shared-catalog)).
- **Integration adapters** — folder + manifest + adapter class в integration-service.

**Защо.** Това е core extensibility requirement-ът на системата: agents и tools трябва да
могат да се добавят от developer (или някой ден да се generated-ват), без platform knowledge.
Open/closed principle, преведен в operations: open for extension, closed for modification.

**Компромис.** Indirection при startup ("откъде дойде този route?") — mitigate-нато чрез
logging на всяко discovered/skipped extension — и нуждата manifest schema да остане стабилна,
защото е public contract.

### 4.3 Declarative manifests (configuration като contract)

**Какво представлява.** *Behaviour envelope* на extension — model, allowed tools, RAG
namespaces, channels, prompts — е data, не code. Runtime чете manifest-а и wiring-ва/ограничава
extension-а съответно.

**Къде.** Agent `manifest.yaml` ([03 §2](./03-agent-platform.md#2-anatomy-of-an-agent-folder));
integration adapter manifests; registry templates (business-level instance на същата идея —
registry се дефинира чрез declarative column specs с canonical roles, не чрез code).

**Защо.** Data може да се review-ва, diff-ва, validate-ва (Pydantic) и редактира без
programming. Повечето behaviour changes (дайте tool на agent, сменете model-а му, scope-нете
RAG-а му) стават едноредови YAML edits.

**Компромис.** Manifests могат тихо да drift-нат от реалността, ако runtime не ги validate-ва
строго — затова discovery time е parse-into-Pydantic-or-skip.

### 4.4 Template Method (default agent graph)

**Какво представлява.** Base implementation дефинира skeleton-а на алгоритъм; specializations
override-ват само различните стъпки. Тук: `build_default_graph()` дава целия
load-context → ReAct loop → approval → answer pipeline, а `default_nodes()` expose-ва
стъпките му като reusable LangGraph nodes, които custom `graph.py` може да recompose-ва.

**Къде.** [03 §1 Pillar 2 and §2](./03-agent-platform.md).

**Защо.** Marginal cost на нов agent е само неговото *unique* behaviour — manifest-only
agent ship-ва zero code; custom agent пише един planning node, не цял loop.

**Компромис.** Default graph е shared dependency: промени в него засягат всеки manifest-only
agent, затова се нуждае от собствени regression tests.

---

## 5. Agentic patterns

### 5.1 ReAct (reason + act tool loop)

**Какво представлява.** Model-ът редува reasoning и tool calls: вижда tool specs, иска call,
runtime го изпълнява, добавя резултата към conversation-а и извиква model-а отново — докато
model-ът отговори без да иска tools.

**Къде.** Цикълът `call_model ⇄ run_tools` в default graph
([03 §1](./03-agent-platform.md)); iteration-capped, за да се предотвратят runaway loops.

**Защо.** Това е най-простият agent pattern, който обработва open-ended business questions
("draft an offer for client X") — и съвпада с това, което ръчно изграденият loop на монолита
вече правеше, минимизирайки behavioral migration risk.

**Компромис.** Latency и token cost растат с всяка iteration; tools трябва да са проектирани
така, че common workflows да се решават с малко calls (например upfront context loading,
per-turn dedupe).

### 5.2 Human-in-the-loop чрез durable interrupts

**Какво представлява.** Преди всяко state-changing action graph-ът спира на checkpoint,
persist-нат в Postgres, показва proposed action на човек и resume-ва с неговото решение —
оцелявайки при disconnects, restarts и channel switches.

**Къде.** `await_approval` interrupt node и `/resume` endpoint
([01 key flow 2](./01-architecture-overview.md), [03 §1 Pillar 5](./03-agent-platform.md)).

**Защо.** ERP agent, който пише registries, изпраща emails и генерира invoices, трябва да е
*provably* неспособен да действа без consent. Монолитът implemented-ваше approval като
in-stream pause (губи се при disconnect); checkpointed interrupts правят consent durable.

**Компромис.** Checkpoint storage и малко по-сложен client protocol (chat може да завърши с
`approval_required`, не с `done`).

### 5.3 Capability allow-listing (least privilege за agents)

**Какво представлява.** Това, че tool съществува в catalog-а, не дава нищо: manifest-ът на
всеки agent изброява точно кои tools model-ът може да вижда и вика, а read/write classification
се enforce-ва от runtime-а (write ⇒ interrupt), не от prompt wording.

**Къде.** `tools:` allow-list и `kind="read"|"write"` enforcement
([03 §4](./03-agent-platform.md#4-tools-the-shared-catalog)); safety guards (host
allow-lists, tenant scoping) живеят вътре в tools, така че никой agent не може да ги забрави.

**Защо.** Prompt-level restrictions са advisory; capability-level restrictions са structural.
Това е security boundary на agent platform.

**Компромис.** Добавянето на capability е умишлено two-step (catalog + manifest), което е
леко friction — by design.

### 5.4 Centralized model access (Model Gateway)

**Какво представлява.** Една услуга притежава всички LLM provider credentials и expose-ва
uniform completion/embedding API; всеки model call в platform-ата минава през нея и emit-ва
metering event като side effect на самия call.

**Къде.** model-gateway ([02](./02-service-catalog.md)); `token.usage` events се consume-ват от
billing-service.

**Защо.** Три структурни печалби: provider switching е невидим за callers; AI kill switch
и budgets действат на едно място; и **billing cannot be bypassed by a forgotten callsite** —
монолитът metering-ваше per-callsite, което е точно как стават leaks.

**Компромис.** Extra hop по най-горещия path (mitigated чрез streaming passthrough и
co-location), а gateway-ът става critical infrastructure — replicate-нете го рано.

---

## 6. Pattern-to-document map

| Pattern | Дефиниран чрез | Основен документ |
|---|---|---|
| Microservices / bounded contexts | service boundaries | [02](./02-service-catalog.md) |
| API Gateway | edge service | [01 §5](./01-architecture-overview.md) |
| Database-per-Service | data ownership rules | [01 §2](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Multi-tenancy + RLS | tenant scoping | [01 §6](./01-architecture-overview.md) |
| Hexagonal (Ports & Adapters) | service internals | [02](./02-service-catalog.md), [03 §5](./03-agent-platform.md) |
| Repository + DTO separation | adapters layer | [03 §5](./03-agent-platform.md) |
| DI / composition root | `deps.py` | [02](./02-service-catalog.md) |
| Sync HTTP + context propagation | call matrix | [01 §5–6](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Thin events + idempotent consumers | Redis Streams topics | [01 §7](./01-architecture-overview.md) |
| Private job queues | per-service workers | [01 §7](./01-architecture-overview.md), [02](./02-service-catalog.md) |
| Strangler Fig | migration phases | [05 §5](./05-migration-pros-and-cons.md) |
| Plugin discovery / convention over config | agents, tools, adapters | [03 §1](./03-agent-platform.md) |
| Declarative manifests | `manifest.yaml` | [03 §2](./03-agent-platform.md) |
| Template Method | default graph | [03 §1–2](./03-agent-platform.md) |
| ReAct loop | default graph | [03 §1](./03-agent-platform.md) |
| Human-in-the-loop interrupts | approval flow | [01 flow 2](./01-architecture-overview.md), [03](./03-agent-platform.md) |
| Capability allow-listing | tool catalog | [03 §4](./03-agent-platform.md) |
| Centralized model access | model-gateway | [02](./02-service-catalog.md) |
