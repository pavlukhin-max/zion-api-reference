# CLAUDE.md — инструкции для Claude Code

## Что это за проект

Публичный API reference сайт для криптоплатёжного шлюза **Zion**. Pure-static — никакого backend, никакого build-step, никакого Node.js runtime. Исходник — `openapi.yaml`, UI — Scalar (CDN), хостинг — Cloudflare Workers Static Assets с native Git integration.

**Production:** https://reference-v2.zionpayment.com
**Workers.dev URL:** https://zion-api-reference.avery-tech2170.workers.dev
**GitHub:** https://github.com/pavlukhin-max/zion-api-reference

## Стек

- **Scalar** — веб-компонент для интерактивного OpenAPI viewer (подключается через CDN `@scalar/api-reference`).
- **OpenAPI 3.1** — стандарт для описания REST API.
- **Cloudflare Workers Static Assets** — статический хостинг с глобальным CDN (унифицированная модель CF, пришла на смену классическому Pages).
- **CF native Git integration** — деплой автоматический по push в `main`, GitHub Actions не используется.

**Чего НЕ будет в этом проекте:**
- Node.js / npm runtime для обслуживания сайта.
- React, Vue, Svelte, Next или любые JS-фреймворки.
- Webpack, Vite, Rollup или другие build-инструменты.
- Backend, база данных, сервер обработки запросов.
- GitHub Actions workflow для деплоя (CF делает это сам).

## Структура источников

```
openapi.yaml              — Source of truth, все эндпоинты + схемы (OpenAPI 3.1)
index.html                — Единственный HTML, подключает Scalar через CDN
guides/                   — Markdown-гайды (getting-started, webhooks, errors) — Phase 3
public/                   — Статика: favicon.svg, favicon.ico, logo.svg
docs/adr/                 — Architecture Decision Records (001 стек, 002 структура openapi)
README.md                 — Быстрый старт
ARCHITECTURE.md           — Архитектура, пайплайны, компоненты
```

## Локальный запуск

**Python (встроенный):**
```bash
cd ~/ClaudeProjects/zion_api_reference/
python3 -m http.server 8000
# → открыть http://localhost:8000
```

**Node.js (быстрее):**
```bash
cd ~/ClaudeProjects/zion_api_reference/
npx serve .
# → открыть http://localhost:3000
```

Оба варианта отдают `index.html` на `/` и `openapi.yaml` на `/openapi.yaml`. Scalar в браузере fetch'ит YAML и рендерит UI.

## Update flow

1. **Правишь `openapi.yaml`** — добавляешь новый эндпоинт, меняешь параметры, обновляешь схемы.
2. **Или правишь `guides/*.md`** — пишешь гайды.
3. **Коммитишь и pushишь в `main`.**
4. **Cloudflare сам клонирует репо** и публикует через Git integration.
5. **За ~30 секунд изменения видны на production** (reference-v2.zionpayment.com).

Никакого локального build, никакого publish-скрипта, никакого CI — всё автоматически.

## Что НЕ делать в этом проекте

### ❌ Архитектура и стек

- **Не добавляй backend** — Express, FastAPI, Fastify, никакие серверные фреймворки.
- **Не добавляй Node.js runtime** — этот сайт не требует выполнения кода на сервере.
- **Не добавляй React / Vue / Svelte / Next** — никаких JS-фреймворков для UI.
- **Не вводи build-step** — webpack, Vite, Rollup, ParcelJS не нужны. Scalar работает в браузере.
- **Не создавай `package.json`** без явной необходимости. Если Scalar когда-то потребует vendor-копию — тогда только через обсуждение с пользователем.
- **Не возвращай GitHub Actions** для деплоя — это пробовали и отказались в пользу CF native Git integration (см. ADR-001 amendment 2026-05-26).

### ❌ Управление файлами

- **Не делай `git clean -fd`, `git clean -fdx`, `git reset --hard`** — риск потерять файлы.
- **Не делай `git rm --cached -r .`** или массовые `git reset` без явного разрешения.
- **Не делай `git push --force`** — рассинхронизирует ветки.
- Если в untracked мусор (node_modules, .env, .next) — скажи пользователю, не удаляй сам.

### ❌ Запись в `.claude/`

- На `.md`-файлы внутри `.claude/` (tasks, memory) — **только через `~/.claude/scripts/hydra-ticket.sh`**.
- На `.md` в корне проекта (README, ARCHITECTURE, CLAUDE, docs/adr) — обычный Write/Edit инструмент ОК.
- На `.ts`, `.json`, `.yaml` — без ограничений (гейт только на `.md` в `.claude/`).

## Брендинг

Применён в Phase 2 — реальные ассеты из `~/ClaudeProjects/_zion_repo/zion/`:

- **Logo:** `public/logo.svg` (источник `frontend/libs/assets/src/svg/logo.svg`)
- **Favicon:** `public/favicon.svg` + `public/favicon.ico`
- **Primary color:** `#09BA69` (зелёный Zion, источник `frontend/libs/theme/ui/src/config/variable/colors.ts`)
- **Accent/danger:** `#F43F5E`
- **Font:** Montserrat через Google Fonts (weights 400/500/600/700)

CSS-переменные Scalar в `index.html` (light + dark mode) уже настроены под Zion. Менять только если придёт новый brand guideline.

## Будущие фазы (короткий контекст)

- ✅ **Phase 0** — скаффолд.
- ✅ **Phase 1** — наполнение `openapi.yaml` (54 paths, 64 schemas, 2 webhooks).
- ✅ **Phase 2** — GitHub-репо, Cloudflare Workers Static Assets, custom domain.
- **Phase 3 — гайды** — наполнить `guides/` (getting-started, webhooks, idempotency, errors, code samples).
- **Phase 4 — watcher** — добавить в helper-bot еженедельную проверку согласованности реального API с `openapi.yaml`.

## Стиль работы

- **Язык:** все тексты, комментарии, коммиты — на русском. Технические идентификаторы (Scalar, OpenAPI, endpoint paths) — на английском.
- **Коммиты:** Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`) приветствуются, но не обязательны.
- **Авторство:** без `Co-Authored-By: Claude ...` trailer'ов.
- **Проверки:** перед `git add .` — смотри `git status`, убедись что в staged не попал мусор (node_modules, .env, .DS_Store).

## Когда вызывать пользователя

- Если нужно выбрать между несколькими архитектурными вариантами.
- Если нужны реальные данные для `openapi.yaml` (версия API, контакт, описание).
- Если что-то требует backend / JS-фреймворка / build-step.
- Если нужно делать деструктивные git-операции (reset, clean, force-push).
- Если неясно, входит ли задача в scope этого проекта.

## Ссылки

- [README.md](./README.md) — как запустить и обновить.
- [ARCHITECTURE.md](./ARCHITECTURE.md) — пайплайн, компоненты, что не входит в scope.
- [docs/adr/001-stack-scalar-cf-pages.md](./docs/adr/001-stack-scalar-cf-pages.md) — почему выбран этот стек.
- [docs/adr/002-openapi-structure.md](./docs/adr/002-openapi-structure.md) — структура openapi.yaml.
