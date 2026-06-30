# DevOps Handoff — сървър + инсталация

Какво да осигурим и инсталираме, за да работи платформата. Пълни детайли: [devops-handoff.bg.md](./devops-handoff.bg.md).

---

## Какво пускаме
- **Един Docker image** за всички app services; променлива `SERVICES` определя кои се пускат във всеки контейнер.
- Фаза 1 = **един app host** с Docker Compose. Не е нужен Kubernetes.
- TLS приключва на **gateway** (единственият публичен контейнер); всичко останало е само вътрешно.

---

## Сървъри за осигуряване

### 1. App host (1 сървър)
Пуска gateway + приложните контейнери + stateless енджините.

| Ресурс | Спецификация |
|---|---|
| vCPU | 8–12 (мин. 4) |
| RAM | 24–32 GB |
| Диск | 100+ GB SSD |
| ОС | Ubuntu LTS (модерен Linux) |
| GPU | Не е нужен (облачни LLM) |

### 2. Данни (отделен хост или managed инстанции — препоръчително)

| Компонент | Спецификация |
|---|---|
| PostgreSQL 16 | 4 vCPU / 16 GB / 100 GB SSD; бекъпи + PITR. Managed е за предпочитане. |
| TimescaleDB (IoT, отделен клъстер) | Postgres 16 + Timescale ext.; 4 vCPU / 16 GB / SSD |
| Redis 7 | 1–2 vCPU / 4 GB; AOF включен + реплика |
| Qdrant | 2 vCPU / 4–8 GB; SSD том |
| S3 / MinIO | Managed S3 в прод, или MinIO хост с достатъчно диск + versioning |

---

## Какво да инсталираме

### На приложния хост
- **Docker Engine + Docker Compose v2** (на практика само това е нужно — app-ът се доставя като един image).

### Хранилища за данни (managed service или самостоятелна инсталация)
- **PostgreSQL 16** (споделен клъстер, по една БД + роля на service)
- **TimescaleDB** (Postgres 16 + Timescale разширение) — отделен клъстер за IoT
- **Redis 7** (AOF включен, + реплика в прод)
- **Qdrant** (векторно хранилище)
- **S3 или MinIO** (обектно хранилище, отделен bucket на service)

### Self-hosted енджини (като контейнери във вътрешната мрежа, не публични)
- **LiteLLM** (достъп до LLM) · **Carbone** (рендиране на PDF/DOCX) · **Unstructured** (OCR/парсване) · **Node-RED** (IoT ingestion edge)
- Stateful (нуждаят се от собствени pg/redis/том, бекъпвайте ги): **Nango**, **Documenso**, **Authentik**
- DB-тата на тези external services живеят в data plane (managed или отделен DB host), с отделни DB + credentials за всяка service.
- Опционални, само при нужда: **Ollama** (локален LLM, нужен е GPU хост), **n8n**

---

## Задължителна конфигурация / операции
- **Цялата конфигурация през променливи на средата**; прод тайните идват от secret manager (никога в кода/image-ите).
- Всяка service използва **собствена БД роля / `DATABASE_URL`**.
- Health: `GET /health` (liveness), `GET /ready` (пропуска трафика).
- **Бекъпи:** по една БД Postgres dump + PITR, Redis AOF + реплика, S3/MinIO versioning, и бекъп на БД-тата на Nango/Documenso/Authentik.

---

## Скалиране по-късно (само конфигурация, същият image)
Добавяне на gateway реплики → ai-plane реплики → отделяне на най-натоварената service → отделяне на останалите групи. Без преархитектуриране.
