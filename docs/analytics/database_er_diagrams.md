# Модель данных и ER-диаграммы микросервисов — Мобильный маркетплейс

**Проект:** Мобильный маркетплейс  
**База данных:** PostgreSQL 16+ (Архитектурный паттерн *Database-per-Service* на уровне логических схем: `auth`, `media`, `catalog`, `commerce`)  
**Первичные ключи:** `UUID` (исключает коллизии между распределёнными сервисами и предсказуемый перебор ID)  
**Денежный тип:** `BIGINT` (суммы в бонусах хранятся целыми числами, исключая погрешности чисел с плавающей точкой)

---

## 1. Схема `auth` (Auth & Profile Service)

Отвечает за учётные записи, сессии (JWT / HttpOnly cookie) и три типа профилей (Покупатель, Продавец, Администратор).

```mermaid
erDiagram
    users ||--o{ user_roles : has
    users ||--o{ refresh_tokens : owns
    users ||--|| buyer_profiles : "1:1 (создаётся сразу)"
    users ||--o| seller_profiles : "1:0..1 (через анкету)"
    users ||--o| admin_profiles : "1:0..1 (только миграцией)"

    users {
        uuid id PK
        varchar email UK "not null"
        varchar password_hash "not null"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    user_roles {
        uuid user_id FK, PK
        varchar role PK "ROLE_BUYER, ROLE_SELLER, ROLE_ADMIN"
    }

    refresh_tokens {
        uuid id PK
        uuid user_id FK "not null"
        varchar token_hash "not null, хэш токена"
        boolean revoked "not null, default false"
        timestamptz expires_at "not null"
        timestamptz created_at "not null"
    }

    buyer_profiles {
        uuid id PK
        uuid user_id FK, UK "not null"
        varchar full_name "nullable"
        varchar phone "nullable"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    seller_profiles {
        uuid id PK
        uuid user_id FK, UK "not null"
        varchar company_name "not null"
        varchar inn "not null, 10 или 12 цифр, неизменяемый"
        text description "nullable"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    admin_profiles {
        uuid id PK
        uuid user_id FK, UK "not null"
        varchar department "nullable, отдел сотрудника"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }
```

### Ключевые инварианты схемы `auth`:

1. **R-12:** `buyer_profiles` создаётся строго в одной транзакции при регистрации `users`. У каждого пользователя гарантированно есть `buyer_id`.
2. **US-1.2:** Поле `inn` валидируется регулярным выражением (`^\d{10}$|^\d{12}$`) и защищено от изменения после создания профиля.
3. **Безопасность (PR-05):** `admin_profiles` создаётся только административной процедурой вне публичного API.
4. **Logout и ротация токенов:** Таблица `refresh_tokens` позволяет отозвать сессию при выходе (`POST /auth/logout`) установкой `revoked = true`.

---

## 2. Схема `media` (Media Service)

Изолированное хранилище метаданных файлов. Поддерживает паттерн прямой загрузки в MinIO через **Presigned URLs** (бэкенд не нагружается бинарным трафиком).

```mermaid
erDiagram
    media_files {
        uuid id PK
        uuid uploader_user_id "not null, ID загрузившего"
        varchar original_filename "not null"
        varchar storage_key "not null, путь в S3 bucket"
        varchar bucket_name "not null"
        varchar content_type "image/jpeg, image/png, image/webp"
        bigint size_bytes "not null"
        text public_url "not null, URL для раздачи"
        varchar status "not null, PENDING / CONFIRMED"
        timestamptz created_at "not null"
        timestamptz confirmed_at "nullable"
    }
```

### Жизненный цикл медиафайла:

- `PENDING`**:** Выдана ссылка `uploadUrl` для прямого `PUT` клиентом в MinIO (TTL ссылки 15 минут).
- `CONFIRMED`**:** Клиент вызвал `POST /api/v1/media/{id}/confirm`, сервис проверил наличие файла в S3 (`HeadObject`) и подтвердил готовность к использованию в карточке товара.

---

## 3. Схема `catalog` (Catalog & Search Service)

Каталог товаров с категориями (табы из Figma) и защитой истории заказов.

```mermaid
erDiagram
    products ||--o{ product_images : has

    products {
        uuid id PK
        uuid seller_id "not null, логическая связь с auth.seller_profiles.id"
        varchar title "not null, min 1 символ"
        varchar category "not null, таб из Figma (electronics, books, clothing...)"
        text description "nullable"
        bigint price "not null, CHECK (price > 0)"
        varchar status "not null (ACTIVE, HIDDEN, ARCHIVED)"
        tsvector search_vector "полнотекстовый поиск (FTS)"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    product_images {
        uuid product_id FK, PK
        uuid media_id PK "логическая связь с media.media_files.id"
        int sort_order "not null, порядок показа (0, 1...)"
    }
```

---

## 4. Схема `commerce` (Commerce Service: Cart, Wallet, Orders)

Финансовый контур маркетплейса: кошелёк с журналом транзакций, корзина, заказы со снимками и защита от повторов.

```mermaid
erDiagram
    wallets ||--o{ bonus_transactions : records
    orders ||--|{ order_items : contains
    cart_items }o--|| wallets : "по buyer_id"

    wallets {
        uuid buyer_id PK "логическая связь с auth.buyer_profiles.id"
        bigint balance "not null, default 0, CHECK (balance >= 0)"
        bigint version "not null, default 0 (optimistic lock)"
        timestamptz updated_at "not null"
    }

    bonus_transactions {
        uuid id PK
        uuid buyer_id "not null"
        bigint amount "not null (+ пополнение/возврат, - списание)"
        varchar operation_type "ADMIN_TOPUP, ORDER_PAYMENT, ORDER_REFUND"
        uuid order_id "nullable, ID заказа"
        uuid admin_id "nullable, ID профиля админа"
        text reason "not null (причина операции)"
        varchar idempotency_key UK "nullable, защита от дублей"
        timestamptz created_at "not null"
    }

    cart_items {
        uuid id PK
        uuid buyer_id "not null"
        uuid product_id "not null"
        uuid seller_id "not null, денормализация для быстрой анти-самопокупки"
        int quantity "not null, CHECK (quantity > 0)"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    orders {
        uuid id PK
        varchar order_number UK "человекочитаемый номер (ORD-YYYYMMDD-XXXX)"
        uuid buyer_id "not null"
        uuid seller_id "not null"
        varchar seller_name_snapshot "not null, снимок названия магазина"
        bigint total_amount "not null, общая сумма заказа"
        varchar status "not null (PAID, IN_PROGRESS, COMPLETED, CANCELLED)"
        timestamptz created_at "not null"
        timestamptz updated_at "not null"
    }

    order_items {
        uuid id PK
        uuid order_id FK "not null"
        uuid product_id "not null"
        varchar title_snapshot "not null, снимок названия товара"
        bigint price_snapshot "not null, снимок цены товара за 1 шт."
        varchar image_url_snapshot "nullable, снимок превью картинки"
        int quantity "not null, CHECK (quantity > 0)"
        bigint subtotal "not null (price_snapshot * quantity)"
    }

    idempotency_keys {
        varchar key PK "UUID ключа из заголовка Idempotency-Key"
        uuid buyer_id "not null"
        varchar operation "CHECKOUT, TOPUP"
        text response_body "кэшированный JSON ответа"
        int response_status "HTTP статус"
        timestamptz created_at "not null"
    }
```

### Защита бизнес-инвариантов в `commerce`:

1. **R-14 (Инвариант баланса):** Ограничение `CHECK (balance >= 0)`. Списание/начисление и создание записи в `bonus_transactions` выполняются строго в одной транзакции:
   `balance = sum(bonus_transactions.amount)`
2. **R-02, US-5.3 AC 5 (Неизменяемость истории):** В `order_items` сохраняются снимки `title_snapshot`, `price_snapshot`, `image_url_snapshot`, а в `orders` — `seller_name_snapshot`. Последующие правки в каталоге не меняют старые чеки.
3. **R-06 (Сумма возврата):** При отмене заказа сумма возврата берётся исключительно из `orders.total_amount` (по снимку), а не по текущей цене товара в каталоге.
4. **US-3.4 (Защита от удаления товара):** Перед физическим удалением товара проверяется наличие записей в `order_items`. Если товар когда-либо заказывался, удаление блокируется (`409 Conflict`), и товар переводится в `ARCHIVED`.
