# Zion API Reference

Публичный API reference сайт для криптоплатёжного шлюза Zion. Построен на [Scalar](https://scalar.com) (веб-компонент для OpenAPI), распределяется через Cloudflare Pages. Источник правды — `openapi.yaml` в корне репо.

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
├── public/                   # Статические файлы: favicon, лого, OG-image
├── docs/adr/                 # Architecture Decision Records
│   ├── 000-template.md       # Шаблон для новых ADR
│   └── 001-stack-scalar-cf-pages.md  # Обоснование стека
├── .github/workflows/deploy.yml  # GitHub Actions для Cloudflare Pages
├── .gitignore
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

Cloudflare Pages автоматически перестроит сайт за ~1–2 минуты после push в `main`.

## Production

Целевой домен: **`reference-v2.zionpayment.com`** (настраивается в Phase 2).

На этапе Phase 0 сайт хостится локально или на staging-домене. После интеграции с Cloudflare Pages и GitHub (Phase 2) каждый push в `main` автоматически деплоится на production.

## Документация

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — общая архитектура, пайплайны, компоненты.
- **[docs/adr/001-stack-scalar-cf-pages.md](./docs/adr/001-stack-scalar-cf-pages.md)** — почему выбраны именно Scalar и Cloudflare Pages.
- **[CLAUDE.md](./CLAUDE.md)** — инструкции для Claude Code в этом проекте.

## Что входит в scope, а что нет

### ✅ В scope этого проекта
- Статический контент (HTML, CSS, OpenAPI YAML)
- OpenAPI 3.1 спецификация (все эндпоинты и схемы)
- Scalar веб-компонент (CDN)
- Markdown-гайды по использованию API
- Cloudflare Pages хостинг
- GitHub Actions auto-deploy

### ❌ Не входит в scope
- Backend / API-сервер
- Node.js runtime для обслуживания сайта
- JavaScript-фреймворки (React, Vue, Svelte, Next)
- Build-step (webpack, Vite, Rollup)
- Database, authentication, payment processing
