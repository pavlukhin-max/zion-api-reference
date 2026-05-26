# Архитектура Zion API Reference

## Обзор

Zion API Reference — pure-static сайт без backend, без runtime, без build-step. Единственный источник правды по API — файл `openapi.yaml`. Scalar (веб-компонент, загружаемый через CDN) рендерит интерактивный UI прямо в браузере пользователя. Статика распределяется глобально через Cloudflare CDN.

**Ключевая идея:** правишь `openapi.yaml` (или `guides/*.md`) → push в GitHub → Cloudflare сам клонирует репо и деплоит → пользователи видят изменения за ~30 секунд.

## Пайплайн

```
┌─────────────────────┐
│  openapi.yaml       │  Source of truth
│  guides/*.md        │  (контент, гайды)
└──────────┬──────────┘
           │
           │ git push origin main
           ▼
┌─────────────────────────────────────────────┐
│ GitHub: pavlukhin-max/zion-api-reference    │
└──────────┬──────────────────────────────────┘
           │
           │ Cloudflare native Git integration
           │ (Workers Static Assets, project zion-api-reference)
           ▼
┌──────────────────────────────────────┐
│ Cloudflare Workers (static assets)   │
│ — clone → upload → distribute        │
└──────────┬───────────────────────────┘
           │
           │ Global CDN
           ▼
┌──────────────────────────────────────┐
│ reference-v2.zionpayment.com         │
│ zion-api-reference.avery-tech2170    │
│   .workers.dev (auto-generated)      │
└──────────────────────────────────────┘
```

## Компоненты

### `openapi.yaml`

OpenAPI 3.1 спецификация — единственный источник правды по API. Содержит:

- **Info section** — название, версия, описание (источник: swagger/description.html из основного Zion-бэкенда).
- **Paths** — все эндпоинты (`/health`, `/transactions`, `/webhooks`, и т.д.) с методами, параметрами, примерами.
- **Components** — переиспользуемые схемы (Transaction, Webhook, Error).
- **Security** — описание способов аутентификации (API key, OAuth, и т.д.).

Scalar парсит YAML и рендерит в интерактивный веб-интерфейс. Изменения в `openapi.yaml` сразу видны в браузере (после refresh).

### `index.html`

Единственный HTML-файл. Содержит:

- DOCTYPE, head с meta-тегами (title, description, OG).
- `<script>` тег, который подключает Scalar из CDN (`@scalar/api-reference`).
- Scalar инициализируется с путём к `./openapi.yaml` и конфигом (брендинг, лого, цвета).
- CSS-переменные для брендинга: primary `#09BA69` (зелёный Zion), Montserrat через Google Fonts.

```html
<script
  id="api-reference"
  data-url="./openapi.yaml"
></script>
<script src="https://cdn.jsdelivr.net/npm/@scalar/api-reference"></script>
```

Никакого Node.js, никакого build-step — это чистый HTML + CDN.

### `guides/`

Директория для markdown-файлов с дополнительной информацией:

- `getting-started.md` — как начать работать с API.
- `webhooks.md` — описание вебхуков, как их обработать.
- `idempotency.md` — как сделать запросы идемпотентными.
- `errors.md` — каталог ошибок и как их обрабатывать.
- `code-samples.md` — примеры на curl, JavaScript, Python.

Гайды **не** встроены в `openapi.yaml` автоматически. На Phase 3 их можно либо добавить в описания эндпоинтов (как ссылки или `externalDocs`), либо оставить отдельной секцией.

### `public/`

Статические ресурсы:

- `favicon.svg` / `favicon.ico` — иконка вкладки браузера.
- `logo.svg` — логотип Zion.

### Cloudflare деплой

Cloudflare project `zion-api-reference` подключён к GitHub репозиторию через native Git integration (OAuth-привязка через CF dashboard). На каждый push в `main` Cloudflare сам клонирует репо, забирает статику (build command пустой, output `/`) и публикует на edge CDN. GitHub Actions не используется.

### `docs/adr/`

Architecture Decision Records — каталог решений:

- **`000-template.md`** — шаблон для новых ADR.
- **`001-stack-scalar-cf-pages.md`** — почему выбраны Scalar и Cloudflare Pages.

Каждое новое решение (например, «как структурировать гайды», «как добавить примеры кода») документируется отдельным ADR.

## OpenAPI structure

Подробные решения по структуре `openapi.yaml` — см. [docs/adr/002-openapi-structure.md](./docs/adr/002-openapi-structure.md). Короткое содержание:

- **Domain decomposition.** 17 тегов по ресурсам (`Shops`, `Payments`, `Withdrawals`, …) сгруппированных в 6 верхнеуровневых категорий через Scalar-extension `x-tagGroups` (`Getting started`, `Commerce`, `Money out`, `Configuration`, `Analytics`, `Realtime`).
- **Security scheme.** Один глобальный `apiKey` в header `Authorization` с подсказкой про префикс `Token <key>` (это DRF TokenAuthentication, не Bearer/OAuth). Public checkout-flow эндпоинты переопределяют `security: []`.
- **Error envelope.** Две схемы: `Error` (`{detail}`) и `ValidationError` (словарь полей). Универсальные `components/responses` (`BadRequest`, `Unauthorized`, `Forbidden`, `NotFound`, `TooManyRequests`) ссылаются на них через `$ref` — никаких inline-error-схем в эндпоинтах.
- **Pagination.** DRF `LimitOffsetPagination` (`?limit=&offset=`, default 10). Параметры — в `components/parameters`, envelope-схемы `Paginated<X>` — через `allOf` композицию с абстрактной `Page`.
- **Webhooks.** Top-level `webhooks` секция OpenAPI 3.1 + payload schemas (`PaymentWebhookPayload`, etc.). HMAC SHA-1 подпись через header `X-Signature`.
- **Naming.** PascalCase для schemas (`Payment`, `PaginatedPayments`, `PaymentCreateRequest`), snake_case для JSON-properties (зеркало Django сериализаторов).
- **Examples.** Обязательны для каждого requestBody, каждого 200/201 response, основных error responses и webhook payloads.

## Что НЕ входит в scope

Следующие технологии и паттерны **не используются** в этом проекте и не должны добавляться без явного запроса:

- **Backend / API-сервер** — нет Express, FastAPI, Fastify, никаких серверных фреймворков.
- **Node.js runtime** — сайт не требует выполнения кода на сервере.
- **React / Vue / Svelte / Next** — нет JavaScript-фреймворков для UI.
- **Build-step** — нет webpack, Vite, Rollup, ParcelJS. Scalar работает прямо в браузере.
- **CSS-препроцессоры** — нет SASS, Less, PostCSS.
- **Database** — нет PostgreSQL, MongoDB, Redis.
- **Authentication backend** — API-keys генерируются в другом сервисе (Zion-бэкенд).

Если задача требует чего-то из этого списка, это **не** задача для этого проекта — это задача для основного Zion-бэкенда.

## Журнал фаз

### Phase 0 — скаффолд ✅ (2026-05-25)

Каркас проекта: `openapi.yaml` (заглушка), `index.html` (Scalar via CDN), документация, ADR-001.

### Phase 1 — наполнение OpenAPI ✅ (2026-05-25)

54 paths по 13 доменам (Shops, Invoices, Donations, POS, Subscriptions, Payments, Fiat, Withdrawals + OTP/affiliate/mass, Conversions, Members, Tokens, Wallets, IPs, Referral/Affiliate, Stats, Hooks audit), 64 schemas с examples, 2 webhooks (HMAC SHA-1 X-Signature), 4 PUBLIC checkout endpoints. ADR-002.

### Phase 2 — Хостинг ✅ (2026-05-25 / 2026-05-26)

GitHub-репо `pavlukhin-max/zion-api-reference`, Cloudflare Workers Static Assets project `zion-api-reference` с native Git integration (OAuth), custom domain `reference-v2.zionpayment.com`. Брендинг Zion (логотип, favicon, palette `#09BA69`, Montserrat) применён.

### Phase 3 — Гайды (планируется)

1. Наполнить `guides/` содержанием (getting-started, webhooks, idempotency, errors, code samples).
2. Добавить ссылки на гайды в описания эндпоинтов (`externalDocs`).

### Phase 4 — Обслуживание (планируется)

Добавить watcher в `zion_helper_bot`, который еженедельно проверяет расхождения между реальным API и `openapi.yaml`.

## Ссылки

- [README.md](./README.md) — быстрый старт.
- [CLAUDE.md](./CLAUDE.md) — инструкции для Claude Code.
- [docs/adr/001-stack-scalar-cf-pages.md](./docs/adr/001-stack-scalar-cf-pages.md) — обоснование выбора стека.
- [docs/adr/002-openapi-structure.md](./docs/adr/002-openapi-structure.md) — структура `openapi.yaml` (tags, security, errors, pagination, webhooks).

## Журнал решений

- 2026-05-25 — ADR-001: Scalar + Cloudflare Pages + OpenAPI 3.1 как стек публичной reference-документации.
- 2026-05-25 — ADR-002: Структура `openapi.yaml` — 17 tags + x-tagGroups, apiKey-auth, Error/ValidationError envelopes, LimitOffsetPagination, top-level webhooks, PascalCase schemas / snake_case properties.

