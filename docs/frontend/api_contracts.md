# Контракт интеграции Frontend с REST API

Документ описывает правила взаимодействия клиентской части с микросервисами бэкенда маркетплейса «Т-Маркет».

---

## 1. Конфигурация клиента Axios

Все запросы отправляются через единый сконфигурированный инстанс Axios:

```typescript
export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api/v1',
  withCredentials: true, // Передача и приём HttpOnly Cookie (refresh-токена)
  headers: {
    'Content-Type': 'application/json'
  }
});
```

### Заголовки запросов:
* `Authorization: Bearer <accessToken>` — инжектируется автоматически интерцептором из оперативной памяти.
* `Idempotency-Key: <UUID>` — обязателен для мутирующих финансовых операций (оформление заказа, списание бонусов).

---

## 2. Фильтрация и сортировка каталога

Для списочного эндпоинта каталога `GET /api/v1/products` зафиксирован следующий синтаксис:

```http
GET /api/v1/products?f[category]=electronics&f[price]=1000,25000&sort=price,asc
```

### Параметры:
* **`f[category]=<slug>`** — фильтр по категории (`electronics`, `clothes`, etc.).
* **`f[price]=<min>,<max>`** — диапазон стоимости (числа через запятую).
* **`sort=<field>,<direction>`** — сортировка:
  * `sort=price,asc` — сначала дешевле;
  * `sort=price,desc` — сначала дороже;
  * `sort=rating,desc` — по рейтингу;
  * `sort=createdAt,desc` — новинки.

---

## 3. Пагинация: Курсорная (Cursor-based) для бесконечного скролла

Для бесконечной ленты каталога вместо `page/size` (или `limit/offset`) используется **курсорная пагинация**:

```http
GET /api/v1/products?f[category]=electronics&limit=20&cursor=ZXlKaGJHY2lPaUpTVX...
```

### Почему от `offset` отказались в пользу `cursor`:
1. **Исключение дублей и пропусков:** При добавлении новых товаров продавцами во время скролла оффсет сдвигается, и пользователь видит один и тот же товар дважды. Курсор жестко привязан к позиции элемента и гарантирует непрерывную ленту.
2. **Производительность БД:** `OFFSET 50000` заставляет базу сканировать и отбрасывать 50 000 строк. Курсор использует индекс (`WHERE (price, id) > (cursor_price, cursor_id)`) и выполняется за доли миллисекунды на любой глубине ленты.

### Формат ответа сервера:
```json
{
  "items": [
    {
      "id": "prod-1",
      "title": "Беспроводные наушники Pro",
      "price": 4990,
      "category": "electronics",
      "rating": 4.8,
      "sellerId": "seller-uuid-1",
      "imageUrl": "https://s3.t-bank.ru/marketplace/media/prod-1.webp"
    }
  ],
  "nextCursor": "ZXlKaGJHY2lPaUpTVX...",
  "hasMore": true,
  "total": 1420
}
```

---

## 4. Загрузка медиа (MinIO / S3)

* **Эндпоинт:** `POST /api/v1/media/upload`
* **Формат:** `multipart/form-data`
* **Поле файла:** `file`
* **Клиентская предобработка:** Перед отправкой файл конвертируется в формат **WebP** со сжатием `quality: 0.82` и ограничением по ширине/высоте `1200px`.

### Формат успешного ответа:
```json
{
  "id": "media-uuid-123",
  "url": "https://s3.t-bank.ru/marketplace/media/media-uuid-123.webp",
  "size": 184520,
  "contentType": "image/webp"
}
```

---

## 5. Формат ошибок (RFC 7807 Problem Details)

Все сетевые ошибки бэкенда обрабатываются фронтендом в едином стандарте:

```json
{
  "type": "about:blank",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Нельзя оформить заказ на товар собственного магазина",
  "instance": "/api/v1/orders/checkout",
  "timestamp": "2026-11-05T20:00:00Z"
}
```
Фронтенд извлекает поле `detail` и отображает его в качестве понятного уведомления для пользователя.
