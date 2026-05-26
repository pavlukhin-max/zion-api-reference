# Zion API Reference

Публичный API reference сайт для криптоплатёжного шлюза Zion. Построен на [Scalar](https://scalar.com) (веб-компонент для OpenAPI), хостится в Cloudflare. Источник правды — `openapi.yaml` в корне репо.

**Production:** https://reference-v2.zionpayment.com

## Локальный запуск

Выбери один из двух способов:

**Вариант 1: Python (встроенный)**
```bash
cd ~/ClaudeProjects/zion_api_reference/
python3 -m http.server 8000
```
Открыть `http://localhost:8000` в браузере.

**Вариант 2: Node.js (быстрее)**
```bash
cd ~/ClaudeProjects/zion_api_reference/
npx serve .
```
Утилита `serve` запустит сервер на порту 3000 (или ближайшем свободном).

## Структура проекта

```
├── openapi.yaml              # Источник правды по API (OpenAPI 3.1)
├── index.html                # Scalar viewer, подключает Scalar через CDN
├── guides/                   # Markdown-страницы (гайды, примеры) — Phase 3
├── public/                   # Статические файлы: favicon, лого
├── docs/adr/                 # Architecture Decision Records
│   ├── 000-template.md       # Шаблон для новых ADR
│   ├── 001-stack-scalar-cf-pages.md  # Обоснование стека
│   └── 002-openapi-structure.md      # Структура openapi.yaml
├── .gitignore
├── ARCHITECTURE.md           # Архитектура и пайплайн
└── CLAUDE.md                 # Инструкции для Claude Code
```

## Как обновить контент

**API эндпоинты и схемы:** отредактируй `openapi.yaml`, сделай коммит и push в `main`.
```bash
git add openapi.yaml
git commit -m "docs: добавлен эндпоинт /transactions"
git push origin main
```

**Гайды и примеры:** добавь `.md`-файл в `guides/`, обнови `openapi.yaml` если нужно (ссылки на гайды в контексте информации об эндпоинте).
```bash
git add guides/getting-started.md
git commit -m "docs: добавлен guide по аутентификации"
git push origin main
```

Cloudflare Workers с native Git integration автоматически перевыкатит сайт за ~30 секунд после push в `main`.

## Production

Live: **https://reference-v2.zionpayment.com**

Также доступно по auto-generated субдомену Cloudflare: `https://zion-api-reference.avery-tech2170.workers.dev`.

Хостинг: Cloudflare Workers Static Assets (унифицированная модель CF, пришедшая на смену классическому Cloudflare Pages). Project name — `zion-api-reference`, production branch — `main`, build-step отсутствует (raw static).

## Документация

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — общая архитектура, пайплайны, компоненты.
- **[docs/adr/001-stack-scalar-cf-pages.md](./docs/adr/001-stack-scalar-cf-pages.md)** — почему выбраны Scalar и Cloudflare.
- **[docs/adr/002-openapi-structure.md](./docs/adr/002-openapi-structure.md)** — структура `openapi.yaml` (tags, security, errors, pagination, webhooks).
- **[CLAUDE.md](./CLAUDE.md)** — инструкции для Claude Code в этом проекте.

## Что входит в scope, а что нет

### ✅ В scope этого проекта
- Статический контент (HTML, CSS, OpenAPI YAML)
- OpenAPI 3.1 спецификация (все эндпоинты и схемы)
- Scalar веб-компонент (CDN)
- Markdown-гайды по использованию API
- Cloudflare хостинг с native Git integration auto-deploy

### ❌ Не входит в scope
- Backend / API-сервер
- Node.js runtime для обслуживания сайта
- JavaScript-фреймворки (React, Vue, Svelte, Next)
- Build-step (webpack, Vite, Rollup)
- Database, authentication, payment processing
