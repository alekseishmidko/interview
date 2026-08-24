# Cookies и их атрибуты

## Как работают cookies

Сервер устанавливает cookie отдельным заголовком ответа:

```http
Set-Cookie: sessionId=abc123; Path=/; Secure; HttpOnly; SameSite=Lax
```

Браузер сохраняет её и добавляет подходящие cookies в запрос:

```http
Cookie: sessionId=abc123
```

`Set-Cookie` не является обычным объединяемым заголовком: для нескольких cookies сервер отправляет несколько строк `Set-Cookie`.

## `HttpOnly`

Запрещает чтение и изменение cookie через JavaScript API вроде `document.cookie`.

```http
Set-Cookie: session=abc; HttpOnly
```

Cookie продолжает автоматически отправляться серверу. Атрибут снижает риск кражи session cookie при XSS, но XSS всё ещё может отправлять запросы от лица пользователя.

## `Secure`

Cookie отправляется только через защищённое соединение HTTPS, с предусмотренными браузерами исключениями для localhost.

```http
Set-Cookie: session=abc; Secure
```

`Secure` не шифрует значение на диске и не защищает cookie от JavaScript без `HttpOnly`.

## `SameSite`

Управляет отправкой cookie в cross-site-контексте.

### `SameSite=Strict`

Наиболее строго ограничивает cross-site-отправку. Может ухудшить UX при переходе пользователя на сайт по внешней ссылке.

### `SameSite=Lax`

Обычно отправляет cookie при same-site запросах и части безопасных top-level переходов, но ограничивает многие cross-site подзапросы и небезопасные методы.

### `SameSite=None`

Разрешает cross-site-отправку. Требует `Secure`.

```http
Set-Cookie: widgetSession=abc; SameSite=None; Secure
```

`SameSite` помогает от CSRF, но для чувствительных операций могут дополнительно использовать CSRF token и проверку `Origin`.

## `Domain`

Если атрибут отсутствует, cookie является host-only и отправляется только установившему хосту.

```http
Set-Cookie: preference=compact; Domain=example.com
```

Такое значение может быть доступно `example.com` и подходящим поддоменам. Сервер не может установить cookie для произвольного чужого домена.

## `Path`

Ограничивает пути, на которые браузер отправляет cookie:

```http
Set-Cookie: admin=value; Path=/admin
```

`Path` не является надёжной границей безопасности между приложениями одного origin.

## `Expires` и `Max-Age`

Без них cookie считается session cookie.

```http
Set-Cookie: theme=dark; Max-Age=2592000
```

```http
Set-Cookie: theme=dark; Expires=Wed, 21 Oct 2026 07:28:00 GMT
```

`Max-Age` задаёт срок в секундах и при одновременном указании обычно имеет приоритет над `Expires`.

Удаление cookie выполняют с теми же `Domain` и `Path`, устанавливая нулевой срок:

```http
Set-Cookie: session=; Path=/; Max-Age=0
```

## `Partitioned`

Запрашивает partitioned storage для cookie: значение изолируется по top-level site, что помогает ограничить межсайтовое отслеживание при embedded-контенте.

```http
Set-Cookie: widget=abc; Secure; SameSite=None; Partitioned
```

Для `Partitioned` требуется `Secure`; поддержка и ограничения зависят от браузера.

## Префиксы имён

### `__Secure-`

Требует установку из secure context и атрибут `Secure`.

### `__Host-`

Требует:

- `Secure`;
- `Path=/`;
- отсутствие `Domain`.

```http
Set-Cookie: __Host-session=abc; Path=/; Secure; HttpOnly; SameSite=Lax
```

Это хороший базовый формат host-bound session cookie.

## First-party и third-party

First-party cookie используется в контексте сайта, который пользователь открыл. Third-party cookie принадлежит embedded-ресурсу другого site. Браузеры всё сильнее ограничивают third-party cookies, поэтому нельзя строить новую архитектуру на предположении об их универсальной доступности.

## Cookie для серверной сессии

```http
Set-Cookie: __Host-session=opaque-id; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=1800
```

В cookie лучше хранить непрозрачный случайный идентификатор, а состояние сессии — на сервере. Нужны ротация, отзыв, ограниченный TTL и защита endpoints от CSRF/XSS.

## Типичные ошибки

- хранить чувствительное значение без `HttpOnly`;
- передавать cookie без `Secure` в production;
- ставить `SameSite=None` без `Secure`;
- считать `Domain` и `Path` полноценной авторизацией;
- хранить большие данные, увеличивая каждый запрос;
- удалять cookie с несовпадающим `Path`;
- считать, что `HttpOnly` полностью устраняет XSS;
- путать CORS с разрешением браузеру отправлять credentials.

## CORS и credentials

Для cross-origin `fetch` с cookies клиент указывает:

```js
fetch('https://api.example.com/profile', {
  credentials: 'include',
});
```

Сервер должен вернуть подходящие CORS-заголовки, включая конкретный разрешённый origin и `Access-Control-Allow-Credentials: true`. При этом cookie всё равно должна удовлетворять правилам `SameSite`, `Secure`, `Domain` и `Path`.

## Краткий ответ

> `HttpOnly` закрывает cookie от JavaScript, `Secure` ограничивает передачу HTTPS, а `SameSite` управляет cross-site-отправкой и снижает риск CSRF. `Domain` и `Path` определяют область отправки, `Max-Age`/`Expires` — срок жизни. Для host-bound server session часто используют имя `__Host-`, `Path=/`, `Secure`, `HttpOnly` и подходящий `SameSite`.

## Источники

- [Using HTTP cookies — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies)
- [`Set-Cookie` — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)

