# ADR-002: Структура `openapi.yaml` — domain decomposition, security, errors, pagination, webhooks

**Статус:** accepted
**Дата:** 2026-05-25
**Авторы:** hydra-architect (через Максима)

---

## Контекст

Phase 0 проекта `zion_api_reference` закрыт: scaffold (Scalar + Cloudflare Pages), брендинг, git репозиторий (https://github.com/pavlukhin-max/zion-api-reference). На данный момент `openapi.yaml` содержит только заглушку `GET /health`.

В Phase 1 нужно наполнить `openapi.yaml` реальной публичной поверхностью Zion API. Из инвентаризации (`zion_helper_bot/docs/zion_api_endpoints.md`) следует, что Zion-gateway работает в нескольких режимах (`PRIVATE`, `PUBLIC`, `MERCHANT`, `SUPPORT`, `AUTH`, `ADMIN`, `DOC`), но **публично документировать имеет смысл только тот срез, который мерчант использует для интеграции** — то есть `MERCHANT` API + минимум `PUBLIC` (checkout-flow, который дёргает фронт оплаты) + `webhooks` (исходящие от Zion на сторону мерчанта).

Поверхность — ~75–80 операций + ~3–5 webhook payloads. Это слишком большой объём, чтобы писать без архитектурного preplan'а: без согласованных решений по tags, security, error envelope, pagination и naming спецификация быстро превратится в кашу из непоследовательно названных схем и дублирующихся определений ответов.

Источники истины при принятии этих решений:

- `zion_helper_bot/docs/zion_api_endpoints.md` — полный список эндпоинтов + serializers + permissions.
- `zion_helper_bot/docs/zion_db_schema.md` — модели данных (для понимания, какие поля попадают в схемы).
- `_zion_repo/zion/backend/srvs/gateway/core/templates/swagger/description.html` — официальная вводная страница API: формат `Authorization` header, HMAC SHA-1 для webhooks, base URL `https://api.zionpayment.com`.
- Технические настройки DRF: `LimitOffsetPagination`, `PAGE_SIZE=10`, `DjangoFilterBackend + OrderingFilter`.

Решение состоит из нескольких взаимосвязанных частей. Каждая оформлена под-секцией с альтернативами и обоснованием.

---

## Решение — общий каркас

```yaml
openapi: 3.1.0
info: { ... }
servers:
  - url: https://api.zionpayment.com
    description: Production
tags: [Authentication, Shops, Payments, Invoices, Donation Links, POS Links, Subscriptions,
       Withdrawals, Conversions, Fiat Payments, Webhooks (incoming), Members, Tokens,
       Wallets & IPs, Affiliate, Analytics, Health]
x-tagGroups:
  - name: Getting started
    tags: [Authentication, Health]
  - name: Commerce
    tags: [Shops, Invoices, Donation Links, POS Links, Subscriptions, Payments, Fiat Payments]
  - name: Money out
    tags: [Withdrawals, Conversions]
  - name: Configuration
    tags: [Members, Tokens, Wallets & IPs]
  - name: Analytics
    tags: [Affiliate, Analytics]
  - name: Realtime
    tags: [Webhooks (incoming)]
paths: { ... }
webhooks: { ... }    # OpenAPI 3.1 top-level
components:
  securitySchemes: { TokenAuth: ... }
  parameters: { LimitParam, OffsetParam, OrderingParam, CreatedAtAfter, CreatedAtBefore, ShopIdPath }
  schemas: { Error, Page<T>, Shop, Payment, Withdrawal, ... }
  responses: { BadRequest, Unauthorized, Forbidden, NotFound, TooManyRequests }
security:
  - TokenAuth: []
```

---

## a) Domain decomposition в tags

### Рассмотренные варианты

**Вариант 1: один плоский tag на каждый ресурс (Shop, Payment, Invoice, ...) — порядка 18–20 тегов.**
Просто, прямолинейно. Но в Scalar навигация превращается в длинный плоский список, мерчанту тяжело найти "как сделать вывод" — он не знает, что искать сначала `Withdrawals`, потом `Mass withdrawals`, потом `Affiliate withdrawals`.

**Вариант 2: тэги по бизнес-домену + `x-tagGroups` для группировки.**
Тэги достаточно гранулярны, чтобы внутри каждого был обозримый набор операций (3–10), а группировка через `x-tagGroups` (Scalar полностью поддерживает) даёт двухуровневую навигацию: верх — что мерчант хочет сделать (`Commerce`, `Money out`, `Configuration`), низ — конкретный ресурс.

**Вариант 3: tags по техническим режимам gateway (`MERCHANT`, `PUBLIC`).**
Отражает внутреннюю архитектуру Zion. Мерчанту не нужно знать о существовании режимов — для него это implementation detail.

### Решение

**Вариант 2.** Tags: `Authentication`, `Shops`, `Invoices`, `Donation Links`, `POS Links`, `Subscriptions`, `Payments`, `Fiat Payments`, `Withdrawals`, `Conversions`, `Members`, `Tokens`, `Wallets & IPs`, `Affiliate`, `Analytics`, `Webhooks (incoming)`, `Health`.

`x-tagGroups` группирует их в 6 верхнеуровневых категорий (см. каркас выше).

**Ключевая причина:** мерчант ищет API "по задаче", а не "по ресурсу". `x-tagGroups` даёт человеческий entry-point ("я хочу принимать платежи" → `Commerce`), сохраняя гранулярность ресурсов для последующего drill-in.

**Замечание:** `Affiliate` собирает все affiliate-* эндпоинты (deposits, referrals, withdrawals, stats). `Wallets & IPs` объединяет `shop-wallets` и `shop-ips` — у них одинаковый назначение (настройка автовыплат).

---

## b) Security schemes

### Контекст

Из `description.html`:

```
Authorization: Token your-generated-api-key
```

Это **DRF TokenAuthentication** (Django REST Framework стандарт), формально это `apiKey` security scheme в OpenAPI с типом `header` и кастомным префиксом `Token `. В OpenAPI 3.x нет first-class способа выразить "header `Authorization: Token <key>`" — только либо `http` с произвольным `scheme`, либо `apiKey` с указанием имени header'а (в этом случае значение header'а целиком — это `Token your-key`, а не просто `your-key`).

### Рассмотренные варианты

**Вариант 1: `apiKey` в header `Authorization`.**
```yaml
TokenAuth:
  type: apiKey
  in: header
  name: Authorization
  description: "Используйте формат: `Token <ваш-api-key>` (с пробелом). Получить ключ — в дашборде."
```
Плюсы: семантически точно. Минусы: пользователь в Scalar Try-it-out должен вписать полное значение header'а вместе с префиксом `Token `, легко забыть.

**Вариант 2: `http` с custom scheme.**
```yaml
TokenAuth:
  type: http
  scheme: token   # нестандартный scheme
```
Это путаница: HTTP scheme `token` не зарегистрирован в IANA, многие клиенты этого не поймут, codegen может сломаться.

**Вариант 3: `http bearer` с уточнением.**
Семантически неверно — это **не** OAuth Bearer. Если codegen увидит `bearer`, он начнёт ожидать поведение OAuth (refresh tokens, expiry), которого тут нет.

**Вариант 4: Описать через `apiKey` + явная инструкция в description с примером.**
Это вариант 1 + tooling-friendly description. В Scalar добавляем явную подсказку про префикс. В examples всех операций — реальный curl с `Authorization: Token ...`.

### Решение

**Вариант 4 (вариант 1 с богатым description).** Один глобальный `security: [{ TokenAuth: [] }]` для всех эндпоинтов кроме `PUBLIC` checkout-flow (`GET /invoices/{id}/`, `GET /donation-links/{slug}/`, `GET /pos-links/{slug}/`, `GET /payments/{id}/`, `POST /invoices/{invoice_id}/payments/`, и т.д. — там `security: []` (override)).

**Webhooks подписываются отдельным механизмом** (HMAC SHA-1, header `X-Signature`) — это **не** security scheme в OpenAPI смысле (это не auth для входящих запросов в Zion, это подпись исходящих от Zion). Документируется в секции `webhooks` отдельным `parameters` блоком + текст в description, см. под-секцию (f).

**Ключевая причина:** `apiKey` — единственный семантически корректный тип для `Authorization: Token <key>` в OpenAPI 3.x. Любая другая попытка натянуть `http bearer` сломает codegen и введёт пользователя в заблуждение.

---

## c) Error envelope

### Контекст

DRF по умолчанию возвращает разные форматы:

- ValidationError (`400`): `{"field_name": ["error message", ...], "non_field_errors": [...]}` — словарь полей.
- APIException (`400`/`403`/`404`/`500`): `{"detail": "error message"}` — одно поле.
- Throttled (`429`): `{"detail": "Request was throttled. Expected available in X seconds."}`.
- Custom Zion exceptions (например `SubscriptionInactiveError`) — кастомные форматы.

### Рассмотренные варианты

**Вариант 1: RFC 7807 (problem+json).**
Стандарт, хорошая дисциплина, поддерживается некоторыми клиентами. **Но Zion-бэкенд не возвращает problem+json** — это потребовало бы изменения серверного кода. Документировать формат, которого нет на сервере, — обман.

**Вариант 2: Один универсальный `ErrorEnvelope` со всеми возможными полями.**
```yaml
ErrorEnvelope:
  type: object
  properties:
    detail: { type: string, description: "Сообщение об ошибке (для APIException)" }
    code:   { type: string, description: "Машинно-читаемый код ошибки (опционально)" }
    errors: { type: object, additionalProperties: { type: array, items: { type: string } },
              description: "Ошибки валидации по полям (для ValidationError, 400)" }
```
Описывает оба варианта (detail + errors) одной схемой. Минус: для конкретного 404 поле `errors` всегда `null`, что вводит в заблуждение. Плюс: одна схема покрывает всё.

**Вариант 3: Два разных schema — `Error` (detail) и `ValidationError` (errors).**
Более точно отражает реальность. В responses `400` ссылается на `ValidationError`, `401`/`403`/`404`/`429`/`500` — на `Error`.

**Вариант 4: Анонимные inline-схемы в каждом response.**
Дублирование, теряем переиспользуемость.

### Решение

**Вариант 3.** Две схемы:

```yaml
components:
  schemas:
    Error:
      type: object
      required: [detail]
      properties:
        detail: { type: string, example: "Authentication credentials were not provided." }
    ValidationError:
      type: object
      additionalProperties:
        type: array
        items: { type: string }
      example:
        amount: ["Ensure this value is greater than 0."]
        currency: ["This field is required."]
  responses:
    BadRequest:
      description: Validation error
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ValidationError' }
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
          example: { detail: "Authentication credentials were not provided." }
    Forbidden:
      description: Permission denied
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    TooManyRequests:
      description: Rate limit exceeded
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
          example: { detail: "Request was throttled. Expected available in 47 seconds." }
```

Каждый эндпоинт references эти `components/responses/*` через `$ref`. Никаких inline-error-схем.

**Ключевая причина:** Точно отражает реальные DRF-ответы. Codegen генерирует две понятные модели вместо одной размазанной.

---

## d) Pagination convention

### Контекст

Из настроек DRF: `DEFAULT_PAGINATION_CLASS = LimitOffsetPagination`, `PAGE_SIZE = 10`. Это даёт стандартный ответ:

```json
{
  "count": 153,
  "next": "https://api.zionpayment.com/shops/abc/invoices/?limit=10&offset=20",
  "previous": "https://api.zionpayment.com/shops/abc/invoices/?limit=10&offset=0",
  "results": [ ... ]
}
```

Все `list` endpoints поддерживают `?limit=` и `?offset=`.

### Рассмотренные варианты

**Вариант 1: Generic-схема `Page<T>` — построить `PaginatedInvoices`, `PaginatedPayments` для каждого ресурса.**
Каноничный способ выразить generic в OpenAPI 3.x. Минус: дублирование (~20 envelope-схем для 20 list-эндпоинтов).

**Вариант 2: Inline envelope в каждом response.**
Без переиспользования. Худший вариант — поломка консистентности на каждом insert'е.

**Вариант 3: Базовая `Page` + `allOf` подмешивает конкретный `items` тип в каждом эндпоинте.**
```yaml
PaginatedInvoices:
  allOf:
    - $ref: '#/components/schemas/Page'
    - type: object
      properties:
        results: { type: array, items: { $ref: '#/components/schemas/Invoice' } }
```
Компактно, но `Page` имеет абстрактный `results` (тип `array`, items `any`), что не идеально с т.з. codegen.

**Вариант 4: Параметры пагинации — единые в `components/parameters`, envelope-схемы — отдельные на каждый ресурс.**

### Решение

Гибрид варианта 3 и 4:

1. **`components/parameters`:**
   ```yaml
   LimitParam:
     name: limit
     in: query
     schema: { type: integer, minimum: 1, maximum: 100, default: 10 }
     description: "Максимум объектов в ответе. По умолчанию 10."
   OffsetParam:
     name: offset
     in: query
     schema: { type: integer, minimum: 0, default: 0 }
     description: "Сколько объектов пропустить."
   OrderingParam:
     name: ordering
     in: query
     schema: { type: string }
     description: "Поле для сортировки. Префикс `-` для убывания. Например: `-created_at`."
   ```

2. **`components/schemas/Page`** — абстрактная (с `results: { type: array }` без типизации):
   ```yaml
   Page:
     type: object
     required: [count, results]
     properties:
       count: { type: integer, description: "Общее количество объектов." }
       next: { type: string, format: uri, nullable: true }
       previous: { type: string, format: uri, nullable: true }
       results: { type: array }
   ```

3. **На каждый ресурс — свой `Paginated<X>` через `allOf`:**
   ```yaml
   PaginatedInvoices:
     allOf:
       - $ref: '#/components/schemas/Page'
       - type: object
         properties:
           results: { type: array, items: { $ref: '#/components/schemas/Invoice' } }
   ```

В list-эндпоинтах:
```yaml
parameters:
  - $ref: '#/components/parameters/LimitParam'
  - $ref: '#/components/parameters/OffsetParam'
  - $ref: '#/components/parameters/OrderingParam'
responses:
  '200':
    content:
      application/json:
        schema: { $ref: '#/components/schemas/PaginatedInvoices' }
```

**Ключевая причина:** Параметры пагинации одинаковы для всех ~20 list-эндпоинтов → выносим в `components/parameters` (DRY). Envelope-схемы типизированы (codegen-friendly) через `allOf`-композицию.

---

## e) Idempotency

### Контекст

Из инвентаризации эндпоинтов **Zion-бэкенд не реализует `Idempotency-Key` header** и не использует ETag для condition-based requests. Mass-withdrawals, payments, withdrawals не упоминают идемпотентность в коде. Но есть **`reference`** — для invoice, payment, withdrawal это уникальный идентификатор операции, сгенерированный сервером, и клиент часто использует его как анти-дубликат (передаёт `order_id` при создании payment'а и сервер отказывается создавать дубль).

### Решение

**Не вводить fake `Idempotency-Key`** в спецификацию (он не реализован на сервере).

Вместо этого — раздел в `info.description` (через markdown): "Идемпотентность через `order_id` при создании платежа: повторный POST с тем же `order_id` для того же `shop_id` вернёт существующий payment вместо создания дубликата" (если это действительно так — нужно подтвердить в Wave A анализом `PaymentSerializer.create()`).

**Открытый вопрос для уточнения:** действительно ли Zion дедуплицирует по `order_id`? Если нет — описать только наличие поля `reference` в ответе (uuid, который клиент должен сохранить для идемпотентного retry в своей системе).

**Ключевая причина:** Документировать механизм, которого нет на сервере, — обман. Если механизма нет, надо сказать: "идемпотентности на уровне HTTP нет, дедуплицируйте на своей стороне через `reference` из 201 ответа".

---

## f) Webhooks

### Контекст

Zion отправляет HTTP POST на `Hook.webhook_url` мерчанта с JSON-телом и подписью `X-Signature: hmac-sha1(secret, body)` (см. `description.html` + сервис `hooker`). Из `db_schema.md` известна структура payload (поле `core_hook.payload` JSONB): он содержит копию payment-объекта с полями `id`, `status`, `currency`, `amount`, `received_amount`, `reference`, `preliminary_model`, и т.д.

OpenAPI 3.1 ввёл top-level `webhooks` секцию специально для документирования исходящих запросов от API в сторону клиента.

### Рассмотренные варианты

**Вариант 1: Описать как обычные paths (псевдо-эндпоинты в стиле `POST /your-callback-url`).**
Семантически неверно — это не Zion-эндпоинты. Confuses Try-it-out (Scalar попытается дёрнуть несуществующий путь).

**Вариант 2: Top-level `webhooks` секция (OpenAPI 3.1).**
Корректный способ. Scalar полностью поддерживает рендеринг этой секции.

**Вариант 3: Описать в `guides/webhooks.md` и не трогать openapi.yaml.**
Меньше дисциплины, легче пропустить изменения в payload'е. Лишает codegen возможности сгенерировать модели для webhook-payload'ов.

### Решение

**Вариант 2.** В `openapi.yaml`:

```yaml
webhooks:
  paymentStatusChanged:
    post:
      summary: Payment status changed
      description: |
        Zion отправляет этот webhook при изменении статуса платежа.
        Тело — JSON. Подпись — `X-Signature: <hmac-sha1-hex>`,
        где секрет — `Hook.secret` (настраивается в дашборде).
      parameters:
        - name: X-Signature
          in: header
          required: true
          schema: { type: string }
          description: "HMAC SHA-1 от raw body, hex-encoded. Секрет — из настроек магазина."
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/PaymentWebhookPayload' }
      responses:
        '200':
          description: Acknowledged. Zion считает webhook доставленным.
        '4xx':
          description: Zion ретраит до N раз.
        '5xx':
          description: Zion ретраит до N раз.
```

Webhook-типы (минимум на старте):
- `paymentStatusChanged` — payment lifecycle (PENDING → COMPLETED / FAILED / REFUNDED).
- `withdrawalStatusChanged` — withdrawal lifecycle.
- `subscriptionEvent` — события подписки (если применимо — уточнить в Wave F).

Для каждого — отдельная webhook-операция + отдельная schema под payload.

**Ключевая причина:** OpenAPI 3.1 даёт first-class конструкцию для исходящих webhook'ов. Scalar её рендерит. Не использовать её — упускать стандартный способ.

---

## g) Naming conventions для schemas / components

### Решение

| Категория | Правило | Пример |
|---|---|---|
| **Ресурс (response model)** | PascalCase, единственное число, имя сущности без суффикса | `Shop`, `Payment`, `Withdrawal`, `Invoice` |
| **Create request body** | `<Resource>CreateRequest` | `PaymentCreateRequest`, `WithdrawalCreateRequest` |
| **Update request body** | `<Resource>UpdateRequest` (PATCH) | `ShopUpdateRequest` |
| **Action body** | `<Resource><Action>Request` | `PaymentManualCollectRequest`, `WithdrawalRefundRequest` |
| **Envelope для пагинации** | `Paginated<ResourcePlural>` | `PaginatedPayments`, `PaginatedShops` |
| **Webhook payload** | `<Event>WebhookPayload` | `PaymentWebhookPayload` |
| **Errors** | `Error`, `ValidationError` (без `Envelope`-суффикса) | `Error`, `ValidationError` |
| **Embedded value-objects** | PascalCase, отдельная схема (НЕ inline) | `Money`, `ShopBalance`, `TransactionHash` |
| **Enums** | PascalCase, единичное название с суффиксом `Status` / `Type` где уместно | `PaymentStatus`, `TransactionType` |
| **Path parameter** | snake_case как в URL (`{shop_id}`, `{invoice_id}`) | — |
| **Query parameter** | snake_case (зеркало Django filter полей) | `created_at`, `wallet__address`, `order_id` |
| **JSON property** | snake_case (зеркало Django serializer'ов) | `created_at`, `received_amount`, `fee_paid_by_user` |

**Ключевая причина:** Schemas — это потенциальные SDK types (codegen → TypeScript/Python interfaces), они должны выглядеть как идиоматические class-names. JSON-поля же остаются snake_case потому, что **сервер возвращает snake_case** — переименовывать в camelCase в спецификации означает врать о реальном API.

---

## h) Examples

### Контекст

Scalar рендерит `examples` (или одиночный `example`) в Try-it-out и в visual reference. Без примеров мерчант не видит, как реально выглядит запрос/ответ.

### Решение

**Обязательно прикладывать примеры в следующих местах:**

1. **На каждый `requestBody`** — минимум 1 пример (`examples: { default: { ... } }`).
2. **На каждый `200`/`201` response** — минимум 1 пример.
3. **На основные error responses** — пример для `400 ValidationError` (показать формат словаря полей) и для `401 Unauthorized` (показать `{detail: "..."}`). Остальные 4xx могут переиспользовать `examples` через `$ref` на `components/examples/*` или иметь inline-пример.
4. **На security scheme** — в description указать реальный curl с `Authorization: Token <key>`.
5. **На webhooks** — обязательно пример payload'а (это для интегратора главное, что он увидит на своей стороне).

**Где `examples` опциональны:** ресурсы без body (GET-запросы без query) — пример response достаточен.

**Где категорически нельзя без examples:** money amounts (показать формат строки `"10.000000"` vs число), enums (status values), даты (`2026-05-25T12:34:56Z`).

**Ключевая причина:** в финтехе пример — половина документации. Особенно для amount-полей: Zion хранит суммы как строки с фиксированным десятичным форматом — без примера интегратор обязательно попробует прислать `number` и получит непонятную validation-error.

---

## Альтернативы (общая таблица)

| Решение | Отвергнутая альтернатива | Причина |
|---|---|---|
| 6 tag-групп через `x-tagGroups` | Плоский список 18 tags | Мерчант ищет по задаче, не по ресурсу |
| `apiKey` в `Authorization` | `http bearer` | Семантически неверно: это не Bearer / OAuth |
| `Error` + `ValidationError` | RFC 7807 | Сервер не возвращает problem+json |
| `LimitParam` + `OffsetParam` через `$ref` | Cursor-based | DRF возвращает limit/offset, переписывать сервер ради cursor — overkill |
| Не документировать `Idempotency-Key` | Документировать fake-header | Он не реализован на сервере |
| Top-level `webhooks` (OpenAPI 3.1) | Markdown в guides/ | Lost machine-readability, no codegen |
| snake_case JSON-поля | camelCase | Сервер возвращает snake_case |
| PascalCase schema-имена с суффиксами | camelCase | Schemas — это SDK types, нужны идиоматичные имена |

---

## Последствия

**Плюсы:**

- Единая, предсказуемая навигация: мерчант видит `Commerce → Payments` сразу, без блуждания.
- Codegen работает: одна `Error`-схема, одна `ValidationError`, типизированные `Paginated<X>` — TypeScript/Python SDK будет генерироваться без сюрпризов.
- Webhooks — first-class в спецификации, integrators видят их рядом с обычными ручками.
- Низкий cognitive load для следующих coder-волн: правила naming/pagination/errors — выписаны, новых решений по ходу не требуется.
- Спецификация **не врёт** о реальном API: формат auth header'а, формат ошибок, формат сумм — всё как на сервере.

**Минусы / цена:**

- ~20 envelope-схем `Paginated<X>` — много, но они однотипные, генерируются по шаблону.
- Если Zion-бэкенд в будущем переедет на JWT — придётся менять security scheme (мелочь, но требует обновления).
- `Authorization: Token <key>` в Scalar Try-it-out требует от пользователя самому добавить префикс `Token ` — это нужно явно подсказать в security description.

**Что становится сложнее:**

- Любое новое решение по структуре (новый формат идемпотентности, новый тип webhook'ов) должно проходить через дополнение этого ADR или новый ADR-003+.

---

## Threat model

API reference — публичная статика, secret'ы в нём не лежат. Тем не менее:

**Угрозы:**

1. **Опечатка в security scheme description** → интегратор отправит `Authorization: Bearer <key>` вместо `Token <key>` → 401 → плохой DX.
2. **Скриншот реального `Authorization: Token ...`** в примере → утечка api-key (если хост посчитает пример настоящим).
3. **Webhook payload schema** показывает имена полей внутреннего DB-модели → информация для атакующего, какие поля Zion использует. Это **низкий риск** — мерчанты должны это знать, чтобы парсить webhook.

**Митигации:**

1. В security description — явный пример curl с placeholder'ом `<your-api-key>`, в каждом code-sample'е — тоже placeholder.
2. **Никогда** в examples не вставлять реальные api-key / shop-id / wallet-address — только заведомо фейковые UUID и адреса (например, `00000000-0000-0000-0000-000000000000`).
3. Не описывать в webhooks внутренние поля, которые не имеют значения для интегратора (например, internal database IDs воркеров).

**Остаточный риск:** мерчант может неправильно проверить HMAC SHA-1 подпись (взять `sha256` вместо `sha1`, либо использовать кодировку base64 вместо hex). Митигация: явный code-sample в `guides/webhooks.md` (Node.js + Python) на верификацию подписи.

---

## План внедрения (для lead)

Работа разбивается на 7 волн (см. `domain wave plan` в тикете). Краеугольный принцип: **Wave A создаёт каркас, потом waves B–F работают параллельно по доменам, Wave G — финальный smoke-test**.

```
Wave A — components (schemas: shells + Error + ValidationError + Page) +
         components/parameters (Limit/Offset/Ordering + path-параметры) +
         components/responses (BadRequest, Unauthorized, Forbidden, NotFound, TooManyRequests) +
         components/securitySchemes (TokenAuth) +
         tags + x-tagGroups +
         webhooks: { } shell (без payload'ов пока) +
         servers + info обновить
Wave B — Authentication tag (если нужны эндпоинты регистрации/логина в публичной части —
         см. открытые вопросы) + Health (уже есть)
Wave C — Shops domain: top-level + actions (default, balance, estimated-balance,
         affiliate-balance, total-deposits-summary, affiliate-stats) ~9 операций
Wave D — Commerce: Invoices (3) + Donation Links (5) + POS Links (5) +
         Subscriptions (CRUD + invite, 6) + Payments (list/retrieve/create + resolve, 4) +
         Fiat Payments (3 + countries, 4) ~27 операций
Wave E — Money out: Withdrawals (regular 2, with-otp 2, affiliate 2, mass 3) +
         Conversions (2) ~11 операций
Wave F — Configuration & analytics: Members (4) + Tokens (4) + Shop Wallets (4) +
         Shop IPs (4) + Referral links (4) + Affiliate-deposits/referrals/stats (3) +
         Daily/monthly stats (4) + Hooks read-only (2) + Payments-export (1) ~30 операций
Wave G — Webhooks payloads (paymentStatusChanged, withdrawalStatusChanged,
         subscriptionEvent — ~3 типа) + финальный smoke-test (yamllint, openapi-spec-validator,
         локальный Scalar render)
```

Каждый coder в каждой волне работает по правилам этого ADR (naming, pagination, error responses через `$ref`, examples обязательны). Lead перед стартом волн B–F приклеивает к каждому брифу выдержку из секций (g), (c), (d), (h).

---

## Откат

Если в процессе Wave A или Wave B окажется, что какое-то решение не работает (например, `apiKey` Authorization не рендерится в Scalar как ожидается, или `allOf` `Paginated<X>` ломает codegen):

1. Не пытаться чинить на ходу — остановить волну.
2. Сделать issue на конкретное решение из этого ADR.
3. Открыть ADR-003 с superseding decision (`superseded by ADR-003` в шапке этого ADR).
4. Lead перезапускает волну.

Структурные решения в этом ADR не "невозвратные" — `openapi.yaml` пока пустой, переделать каркас в Wave A дёшево. После Wave B–F — дороже (нужно переписывать `$ref`'ы во всех эндпоинтах), но всё ещё возможно скриптом.

---

## Ссылки

- ADR-001: Стек публичного API reference — Scalar + Cloudflare Pages + OpenAPI 3.1
- ARCHITECTURE.md — секция "OpenAPI structure" (добавлена в этом коммите)
- Тикет: TASK-20260525-2244-phase1-openapi-content
- Источник правды по эндпоинтам: `~/ClaudeProjects/zion_helper_bot/docs/zion_api_endpoints.md`
- Источник правды по моделям: `~/ClaudeProjects/zion_helper_bot/docs/zion_db_schema.md`
- Auth/webhooks: `~/ClaudeProjects/_zion_repo/zion/backend/srvs/gateway/core/templates/swagger/description.html`
- OpenAPI 3.1 webhooks spec: https://spec.openapis.org/oas/v3.1.0#fixed-fields-0
- Scalar `x-tagGroups` поддержка: https://github.com/scalar/scalar
