# HTTP-кеширование

## Зачем нужен кеш

HTTP-кеш сохраняет ответ и повторно использует его, уменьшая latency, трафик и нагрузку на сервер.

Виды кешей:

- private browser cache;
- shared proxy/CDN cache;
- managed cache внутри Service Worker — отдельный механизм Cache API.

HTTP cache и Cache API не одно и то же: первым управляют HTTP-семантика и браузер, вторым — код приложения.

## Freshness и validation

Свежий ответ можно использовать без запроса к origin. Устаревший ответ можно провалидировать условным запросом и получить `304 Not Modified` без повторной передачи body.

```text
Fresh → use cached response
Stale → conditional request → 304 or new 200
```

## `Cache-Control`

### `max-age`

```http
Cache-Control: max-age=3600
```

Ответ считается свежим указанное число секунд от момента происхождения с учётом вычисления возраста.

### `s-maxage`

```http
Cache-Control: public, max-age=60, s-maxage=3600
```

Задаёт свежесть для shared cache и переопределяет там `max-age`.

### `public`

Разрешает shared cache хранить ответ, в том числе в случаях, где это иначе было бы ограничено.

### `private`

```http
Cache-Control: private, max-age=60
```

Разрешает private cache, но запрещает shared cache. Подходит для персонализированных ответов, если их вообще безопасно хранить в браузерном кеше.

### `no-cache`

Не означает «не хранить». Ответ можно сохранить, но перед повторным использованием нужно валидировать у origin.

```http
Cache-Control: no-cache
```

### `no-store`

Запрещает кешу сохранять ответ.

```http
Cache-Control: no-store
```

### `must-revalidate`

После устаревания нельзя использовать сохранённый ответ без успешной валидации, если другие правила не разрешают это явно.

### `immutable`

Указывает, что свежий ресурс не изменится. Полезно для versioned assets:

```http
Cache-Control: public, max-age=31536000, immutable
```

### `stale-while-revalidate`

Разрешает временно вернуть устаревший ответ, параллельно обновляя кеш:

```http
Cache-Control: max-age=60, stale-while-revalidate=300
```

## `ETag`

Сервер выдаёт валидатор версии:

```http
ETag: "products-v42"
```

Клиент валидирует:

```http
If-None-Match: "products-v42"
```

Если представление не изменилось:

```http
HTTP/1.1 304 Not Modified
```

Body в `304` не передаётся; клиент использует сохранённый body и обновляет metadata по правилам HTTP.

## Strong и weak ETag

Strong ETag означает побайтово эквивалентное представление для соответствующих сравнений:

```http
ETag: "abc"
```

Weak ETag допускает семантическую эквивалентность без побайтового равенства:

```http
ETag: W/"abc"
```

## `Last-Modified`

```http
Last-Modified: Mon, 24 Aug 2026 10:00:00 GMT
```

Условный запрос:

```http
If-Modified-Since: Mon, 24 Aug 2026 10:00:00 GMT
```

`ETag` обычно точнее: timestamp может иметь ограниченную точность, а несколько изменений могут попасть в один интервал.

## `Vary`

```http
Vary: Accept-Encoding, Accept-Language
```

Сообщает, какие request headers влияют на выбор представления. Кеш должен учитывать их значения.

```http
Vary: *
```

Фактически делает повторное использование ответа другим запросом невозможным.

Нельзя без необходимости добавлять высококардинальные headers вроде полного `User-Agent` или `Cookie`: это дробит кеш и снижает hit ratio.

## Статические assets

Рекомендуемый паттерн:

```text
/assets/app.a19c8e7.js
```

```http
Cache-Control: public, max-age=31536000, immutable
```

При изменении содержимого меняется hash и URL. HTML обычно получает более короткую свежесть или validation, чтобы быстро ссылаться на новую сборку.

## API

Публичный редко меняющийся ответ:

```http
Cache-Control: public, max-age=60, s-maxage=300
ETag: "catalog-v17"
```

Чувствительный ответ:

```http
Cache-Control: no-store
```

Настройки зависят от данных и модели угроз. `private` и `no-store` имеют разную семантику.

## Cache busting

Изменение query string или filename создаёт другой cache key:

```text
/app.js?v=42
/app.42.js
```

Content-hashed filename надёжнее связывает URL с содержимым и удобен для immutable caching.

## Типичные ошибки

- считать `no-cache` запретом хранения;
- кешировать персонализированный ответ как `public`;
- не добавлять `Vary`, когда ответ зависит от request header;
- ставить долгий TTL на неизменяемый URL без механизма invalidation;
- генерировать новый ETag при каждом запросе для неизменившегося ресурса;
- ожидать, что `304` содержит body;
- путать browser cache, CDN cache и application cache;
- не учитывать, что `Set-Cookie` и `Authorization` влияют на правила shared cache.

## Типичные вопросы

### `no-cache` против `no-store`?

`no-cache` разрешает хранение, но требует validation перед использованием. `no-store` запрещает хранение ответа кешем.

### Чем `200` из кеша отличается от `304`?

При свежем кеше сетевого запроса может не быть. `304` приходит после conditional request и сообщает использовать сохранённое body.

### Что лучше: ETag или Last-Modified?

ETag даёт серверу точный идентификатор версии; Last-Modified проще, но ограничен временной точностью. Можно использовать оба.

## Краткий ответ

> HTTP-кеш использует freshness и validation. `max-age` задаёт свежесть, `ETag`/`If-None-Match` и `Last-Modified`/`If-Modified-Since` позволяют получить `304`. `no-cache` требует проверки, а `no-store` запрещает хранение. `private` ограничивает ответ приватным кешем, `s-maxage` управляет shared cache, а `Vary` описывает request headers, влияющие на представление.

## Источник

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)

