# x7-common — споделеното ядро
> Един малък пакет с леки зависимости (`libs/common`, име за импортиране`x7_common`) инсталиран> от всяка услуга. Той притежава скучната, но критична водопроводна инсталация, така че всички 11 услуги да стартират,> регистрирайте, удостоверявайте, извиквайте се взаимно и публикувайте събития по абсолютно същия начин.
**Статус:** 📋 планирано · **Местоположение:**`libs/common`· **Разпространение:**`x7-common` ·
**Внос:**`x7_common`

## Договорът
Две правила правят споделеното ядро ​​безопасно вместо капан за свързване:
1. **Никаква бизнес логика.** Ако дадена функция знае какво е фактура, регистър илиагент е, не му е мястото тук. Само транспорт, видимост, автентичен водопровод имеждусервизни DTO.2. **Допълнителна еволюция.** Всяка услуга зависи от този пакет, така че е важна промянасмяна на 11 услуги. Новите полета/помощниците са добре; преименувания и премахвания се нуждаят от aпрозорец за амортизация.
Услугите не могат **никога** да импортират код на друга услуга —`x7_common`е единственият споделенPython зависимост между тях.
## Оформление на пакета
```
libs/common/
├── x7_common/
│   ├── __init__.py
│   ├── config.py          # CommonSettings — base for every service's Settings
│   ├── auth.py            # AuthContext from gateway headers · service tokens · require_role
│   ├── http.py            # process-wide pooled httpx.AsyncClient
│   ├── middleware.py      # RequestIdMiddleware (read/generate/echo X-Request-Id)
│   ├── logging.py         # structlog JSON setup, request_id/tenant contextvars
│   ├── telemetry.py       # OpenTelemetry init (FastAPI + httpx instrumentation)
│   ├── health.py          # health_router(checks) → /health + /ready
│   ├── lifespan.py        # composable lifespan (httpx pool + service hooks)
│   ├── events.py          # Redis Streams EventBus: publish / consumer groups
│   ├── errors.py          # error envelope + exception handlers (uniform 4xx/5xx shape)
│   ├── pagination.py      # cursor pagination params + response wrapper
│   ├── schemas/           # cross-service DTOs (the wire contracts)
│   │   ├── auth.py        #   AuthContext, Role
│   │   ├── chat.py        #   ChatMessage, ToolSpec, ToolCall, Usage, StreamChunk,
│   │   │                  #   CompletionRequest/Response, EmbeddingsRequest/Response
│   │   ├── retrieval.py   #   Chunk, RetrieveRequest/Response
│   │   └── events.py      #   typed event payloads (TenantCreated, TokenUsage, …)
│   └── testing/
│       └── fixtures.py    # app factory fixture, fake auth headers, event-bus stub
├── tests/
└── pyproject.toml
```

## Ръководство на модула
### `config.py` — `CommonSettings`

Всяка услуга`Settings`подкласове това; цялата конфигурация се управлява от env (12-фактор).
```python
class CommonSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    service_name: str = "x7-service"
    environment: str = "local"
    log_level: str = "INFO"

    # Internal addressing — Docker DNS names; overridden per environment
    gateway_url: str = "http://gateway:8000"
    identity_url: str = "http://identity-service:8000"
    agent_url: str = "http://agent-service:8000"
    model_gateway_url: str = "http://model-gateway:8000"
    knowledge_url: str = "http://knowledge-service:8000"
    registry_url: str = "http://registry-service:8000"
    business_url: str = "http://business-service:8000"
    document_url: str = "http://document-service:8000"
    billing_url: str = "http://billing-service:8000"
    integration_url: str = "http://integration-service:8000"
    platform_url: str = "http://platform-service:8000"

    # Redis (cache, queues, event streams)
    redis_url: str = "redis://redis:6379/0"

    # Service-to-service auth
    service_token_audience: str = ""        # set to the service's own name
    service_token_ttl_seconds: int = 60

    # Telemetry
    otel_enabled: bool = False
    otel_exporter_otlp_endpoint: str | None = None
```

Услугата добавя само свои собствени полета (`database_url`, `stripe_secret`, …) в своя`config.py`.

### `auth.py`— идентификационен водопровод (без правила)
Ключова разлика от библиотека за декодиране на носител: в тази архитектура **шлюзът проверяваJWT** на крайния потребител (RS256 чрез JWKS) и препраща проверени заглавки. Услугите следователно*извличане*, а не *проверка*, идентичност на потребителя — но те **проверяват** токените на услугата, които доказватобаждането дойде от вътрешността на платформата.
```python
@dataclass(frozen=True, slots=True)
class AuthContext:
    user_id: str
    company_id: str          # the active tenant
    roles: list[str]         # roles within that company
    is_platform_admin: bool
    request_id: str

def auth_context(request: Request) -> AuthContext: ...
    # FastAPI dependency — reads X-User-Id / X-Company-Id / X-Roles / X-Request-Id;
    # 401 if missing (a request that bypassed the gateway)

def require_role(*allowed: str) -> Depends: ...
    # require_role("owner", "co-owner") — 403 below threshold; 404 for
    # platform-admin-only routes (ADR-015 behavior)

def require_service_token(audience: str) -> Depends: ...
    # verifies X-Service-Token minted by identity-service (iss/aud/exp);
    # applied to /internal/* routes
```

Механика на сервизния токен — предназначена да държи услугата за самоличност **от пътя на заявка**:клиентският помощник изсича токен веднъж и го кешира до малко преди това`exp`, освежаващовъв фонов режим;`require_service_token`проверява подписа **локално** срещууслугата-токен JWKS на услугата за идентичност (кеширана, със същото`kid`обработка на въртене катопотребителят JWKS). Следователно прекъсването на услугата за самоличност блокира само подновяването на токена – и самоведнъж кешираните токени изтичат — никога трафик по време на полет.
### `http.py`— един обединен`httpx.AsyncClient`на процес
Singleton със разумни изчаквания и ограничения на връзката; инициализирано/затворено от продължителността на живота.Използват се всички адаптери`get_client()`— никой не създава ad-hoc клиенти (прекъсване на връзкатапътят на инструмента на агента е реална цена за забавяне).
### `events.py`— автобусът за събития Redis Streams
Частта, която mango-common няма, изисквана от нашите управлявани от събития потоци([01 §7](../../01-architecture-overview.md#7-asynchronous-work-and-events)):

```python
bus = EventBus(redis_url, service_name)

await bus.publish("token.usage", TokenUsage(tenant_id=..., tokens=..., feature=...))

@bus.consumer("tenant.created", group="registry-service")
async def on_tenant_created(evt: TenantCreated) -> None:
    ...  # idempotent — at-least-once delivery
```

- Въвели полезни товари от`schemas/events.py`(Pydantic-утвърден от двата края).- Групи потребители по услуга; автоматично потвърждение при успех, повторен опит с мъртва буква след Nнеуспехи.- Всяко събитие носи`event_id`, `occurred_at`, `tenant_id`, `request_id`— използват потребителите  `event_id`като техен ключ за идемпотентност.
### `logging.py` + `middleware.py` + `telemetry.py`— наблюдаемост
- structlog JSON линии с`service`, `request_id`, `user_id`, `company_id`обвързан отcontextvar — зададен веднъж от`RequestIdMiddleware` + `auth_context`, присъства на всеки ред.- `RequestIdMiddleware`: прочети`X-Request-Id`(или генериране), свързване, ехо при отговор.- `telemetry.init(app)`: Проследяване на OpenTelemetry за FastAPI + httpx, така че проследяването е започнало вшлюзът продължава през всеки скок.
### `health.py` + `lifespan.py`— равномерна оперативна повърхност
- `health_router(checks={"db": ping_db, "redis": ping_redis})`монтажи`GET /health`
(жизненост, винаги 200) и`GET /ready`(503, ако някое изследване е неуспешно) — договорът DockerCompose / k8s проверките разчитат за всяка услуга.- `make_lifespan(on_startup=[...], on_shutdown=[...])`зарежда httpx пула, телеметрия,и шината на събитията, след което изпълнява специфични за услугата кукички (DB двигател, arq pool).
### `schemas/`— теленните договори
Само DTO, които **пресичат граница на услуга**, живеят тук (форми за чат/завършване, използвани отагент-услуга ↔ модел-шлюз, парчета за извличане, полезни товари за събития,`AuthContext`). DTOизползвано от точно една услуга остава в тази услуга`models/`— напр. сесия/съобщениеDTOs живеят в агентска услуга, тъй като разговорите са вътрешни за тях. Преместване на тип туке умишлен акт: става замразен договор.
### `errors.py` + `pagination.py`— API последователност
- Един плик за грешка (`{"error": {"code", "message", "request_id"}}`) и споделено изключениеманипулатори, така че всички 11 услуги се провалят идентично и интерфейсът обработва една форма.- Параметри/отговор за страниране на курсора, използвани от всяка крайна точка на списък.
### `testing/`— приспособления, които всяка услуга използва повторно
Фабрично приспособление с продължителност на живота,`auth_headers(user, company, roles)`за симулиранеgateway-forwarded identity, in-memory`EventBus`мъниче, потвърждаващо публикувани събития, и aпомощник за еднократна употреба на Postgres (testcontainers).
## Какво умишлено НЕ принадлежи тук
| Изкушаващо за споделяне | Защо остава навън |
|---|---|
| SQLAlchemy база / хранилища | База данни за услуга: всяка услуга притежава своя форма на постоянство |
| Изброявания на домейни (състояние на фактура, роли в регистъра) | Бизнес логика — принадлежи към договора на притежаващата услуга |
| Класове клиенти на услуги (`RegistryClient`, …) | Повикващите дефинират *портове* за това, от което *се* нуждаят (шестоъгълно правило); споделен клиент би свързал всеки потребител с една форма |
| LLM/доставчик SDK обвивки | само адаптери на модел-шлюз |
| Флагове за функции / ключове за изключване | модел-портал и притежаване на услуги |

## Зависимости и инструменти

```toml
[project]
name = "x7-common"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.111", "httpx>=0.27",
  "pydantic>=2.7", "pydantic-settings>=2.3",
  "structlog>=24", "pyjwt>=2.8",
  "redis>=5",                       # event bus + cache
  "opentelemetry-sdk>=1.25",
  "opentelemetry-instrumentation-fastapi>=0.46b0",
  "opentelemetry-instrumentation-httpx>=0.46b0",
]
[project.optional-dependencies]
dev = ["pytest>=8", "pytest-asyncio>=0.23", "ruff>=0.5", "mypy>=1.10", "testcontainers>=4"]
```

Инсталиран от всяка услуга като a`uv`зависимост от пътя на работното пространство(`x7-common = { workspace = true }`), така че промените се взимат локално без публикуване;CI изпълнява общия тестов пакет преди всеки сервизен пакет.
## Контролен списък за внедряване
- [ ] Пакетно скеле + pyproject + uv работно окабеляване- [ ] `CommonSettings`, `http`, `lifespan`, `health`(най-малкото полезно ядро)- [ ] `RequestIdMiddleware`+ настройка на structlog + контекстни променливи- [ ] `AuthContext`зависимост +`require_role`+ токени за услуга (с услуга за самоличност)- [ ] `EventBus`публикуване/консумиране + мъртво писмо + въведени полезни натоварвания за събития- [ ] Обвивка на грешка + обработка на изключения; помощници за пагиниране- [ ] `schemas/`чат + DTO за извличане (първо агент ↔ договор за модел-шлюз)- [ ] `testing/`тела; общ тестов пакет в CI пред сервизните пакети- [ ] OpenTelemetry init е проверено от край до край през шлюза
## Препратки
- [02 — Каталог на услугите (споделен`libs/common`правила)](../../02-service-catalog.md)
- [01 §7 — събития](../../01-architecture-overview.md#7-asynchronous-work-and-events) ·
[01 §10 — междусекторни стандарти](../../01-architecture-overview.md#10-cross-cutting-standards-every-service)
- [06 — шаблони: DI/композиционен корен, тънки събития](../../06-architectural-patterns.md)
