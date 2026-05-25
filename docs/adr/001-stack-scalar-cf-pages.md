# ADR-001: Стек публичного API reference — Scalar + Cloudflare Pages + OpenAPI 3.1

**Статус:** accepted  
**Дата:** 2026-05-25  
**Авторы:** Максим (owner)

---

## Контекст

Нужен публичный API reference сайт для Zion (криптоплатёжный шлюз). Аналог reference.zionpayment.com, но собственный и под полным контролем. Требования:

- Бесплатный или минимально дешёвый хостинг
- Современный UI без ощущения «документация из 2015 года»
- OpenAPI как единственный источник правды (изменение yaml = обновление сайта)
- Минимум build-step'ов — меньше движущихся частей, меньше поломок
- Простой update flow: правка yaml → git push → автодеплой

## Решение

**Scalar** (`@scalar/api-reference`) через CDN как OpenAPI viewer — один `<script>`-тег в `index.html`, рендерит напрямую в браузере из `openapi.yaml`.

**OpenAPI 3.1** в файле `openapi.yaml` в корне репо — источник правды. Все изменения API идут через него.

**Cloudflare Pages** для статического хостинга — авто-деплой из ветки `main` через GitHub Actions (wrangler-action). Кастомный домен `reference-v2.zionpayment.com`.

**Один бранч `main`** без feature branches на статике — для документационного репо ветвление создаёт накладные расходы без пользы.

## Альтернативы

| Вариант | Причина отказа |
|---------|---------------|
| Mintlify | Современный UI, отличная DX, но **$150/мес** для публичных проектов с custom domain — неприемлемо по цене |
| Redocly (redoc) | OSS, но UI выглядит устаревшим, типографика заметно слабее Scalar |
| Stoplight Elements | Приличный вариант, но менее активный development, меньше community momentum |
| Swagger UI | Самый старый игрок, UI уровня 2015 года, тяжёлый bundle |
| GitHub Pages вместо CF Pages | Работает, но CF Pages быстрее (edge CDN), удобнее с кастомными доменами, лучше build limits |
| Vercel вместо CF Pages | Отличный хостинг, но избыточен для pure static — оптимизирован под Next.js/serverless |
| Backend-генератор (Hugo/MkDocs) | Добавляет build-step без реальной выгоды поверх OpenAPI → Scalar |

## Последствия

**Плюсы:**
- Ноль операционных расходов (CF Pages free tier покрывает нагрузку)
- Нет build-step — Scalar рендерит OpenAPI прямо в браузере, нет CI-зависимостей на генерацию HTML
- Один `<script>`-тег для интеграции viewer'а — минимальная сложность
- Легко обновлять: правка `openapi.yaml` → git push → через ~1 мин сайт обновлён
- Современный UI из коробки, тёмная/светлая тема, поиск, Try-it-out
- OpenAPI 3.1 — стандарт для будущей машинной читаемости (codegen, contract testing, SDK-генерация)
- Edge CDN Cloudflare — низкая задержка по всему миру

**Минусы / риски:**
- Рендеринг на клиенте: миллисекунды задержки vs prerender (несущественно для документации)
- Зависимость от CDN jsdelivr/Scalar npm-публикаций — если CDN недоступен, сайт не рендерит viewer. Митигация: при необходимости вендорить скрипт в `/public/`
- OpenAPI 3.1 поддержан не всеми инструментами (некоторые ещё на 3.0) — при интеграции с внешними инструментами проверять совместимость

## Ссылки

- ARCHITECTURE.md — общая архитектура проекта
- README.md — быстрый старт и структура репо
- Тикет: TASK-20260525-2031-scaffold-zion-api-reference
- Scalar docs: https://github.com/scalar/scalar
- Cloudflare Pages docs: https://developers.cloudflare.com/pages/
