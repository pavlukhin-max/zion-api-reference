# Архитектура Zion API Reference

## Обзор

Zion API Reference — pure-static сайт без backend, без runtime, без build-step. Единственный источник правды по API — файл `openapi.yaml`. Scalar (веб-компонент, загружаемый через CDN) рендерит интерактивный UI прямо в браузере пользователя. Статика распределяется глобально через Cloudflare Pages CDN.

**Ключевая идея:** правишь `openapi.yaml` (или `guides/*.md`) → push в GitHub → Cloudflare Pages автоматически деплоит → пользователи видят изменения за ~1–2 минуты.

## Пайплайн

```
┌─────────────────────┐
│  openapi.yaml       │  Source of truth
│  guides/*.md        │  (контент, гайды)
└──────────┬──────────┘
           │
           │ git push origin main
           ▼
┌─────────────────────────────────────┐
│     GitHub Repository               │
│  zion-api-reference (owner/repo)    │
└──────────┬──────────────────────────┘
           │
           │ GitHub Actions workflow
           │ (.github/workflows/deploy.yml)
           ▼
┌──────────────────────────────────────┐
│ Cloudflare Pages                     │
│ (wrangler pages deploy)              │
└──────────┬───────────────────────────┘
           │
           │ Global CDN
           ▼
┌──────────────────────────────────────┐
│ reference-v2.zionpayment.com         │
│ (Production domain — Phase 2)        │
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
- CSS-переменные для брендинга (placeholder, реальные значения — Phase 3).

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

- `favicon.ico` — иконка вкладки браузера.
- `logo.png`, `logo-dark.png` — логотип Zion (для брендинга в Scalar, Phase 3).
- `og-image.png` — preview-изображение для социальных сетей.

### `.github/workflows/deploy.yml`

GitHub Actions workflow, триггеры на push в `main`:

1. Checkout репо.
2. Вызов `wrangler pages deploy` с целевым проектом `zion-api-reference`.
3. Установка секретов (CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID) — Phase 2.

На Phase 0 это заглушка с TODO-комментариями, реальная интеграция — Phase 2.

### `docs/adr/`

Architecture Decision Records — каталог решений:

- **`000-template.md`** — шаблон для новых ADR.
- **`001-stack-scalar-cf-pages.md`** — почему выбраны Scalar и Cloudflare Pages.

Каждое новое решение (например, «как структурировать гайды», «как добавить примеры кода») документируется отдельным ADR.

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

## Развёртывание

### Phase 1 — наполнение OpenAPI

Конвертировать и расширить `openapi.yaml` на основе:
- `~/ClaudeProjects/zion_helper_bot/docs/zion_api_endpoints.md`
- Реальные эндпоинты из Zion-бэкенда.

### Phase 2 — GitHub + Cloudflare Pages

1. Создать GitHub репо `<owner>/zion-api-reference`.
2. Настроить Cloudflare Pages проект, интегрировать с GitHub.
3. Привязать поддомен `reference-v2.zionpayment.com`.
4. Заполнить секреты в GitHub Actions (CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID).

### Phase 3 — Гайды и брендинг

1. Наполнить `guides/` содержанием.
2. Обновить Scalar конфиг в `index.html` с реальными цветами, логотипом.
3. Добавить ссылки на гайды в描述 эндпоинтов (`externalDocs`).

### Phase 4 — Обслуживание

Добавить watcher в `zion_helper_bot`, который еженедельно проверяет расхождения между реальным API и `openapi.yaml`.

## Ссылки

- [README.md](./README.md) — быстрый старт.
- [CLAUDE.md](./CLAUDE.md) — инструкции для Claude Code.
- [docs/adr/001-stack-scalar-cf-pages.md](./docs/adr/001-stack-scalar-cf-pages.md) — обоснование выбора стека.
