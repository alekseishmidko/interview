# `as` vs `satisfies` в TypeScript — подготовка к интервью

> Правильное название оператора — `satisfies`, не `satisfied`.

## Краткий ответ для собеседования

`as` — это **type assertion**: разработчик сообщает компилятору, каким типом нужно считать выражение. Он может изменить тип выражения с точки зрения TypeScript, но не выполняет runtime-проверку и не преобразует значение.

`satisfies` — оператор проверки совместимости: он проверяет, что выражение соответствует указанному типу, но старается сохранить более точный выведенный тип самого выражения.

```ts
type Config = {
  mode: 'development' | 'production';
  port: number;
};

const withAs = {
  mode: 'development',
  port: 3000,
} as Config;

const withSatisfies = {
  mode: 'development',
  port: 3000,
} satisfies Config;
```

В `withAs` выражение рассматривается как `Config`, поэтому `mode` имеет широкий тип `'development' | 'production'`. У `withSatisfies` проверяется совместимость с `Config`, но сохраняется более точная информация о значении — `mode` остаётся `'development'`.

Главная формулировка:

> `as` говорит компилятору «считай это таким типом», а `satisfies` спрашивает компилятор «соответствует ли значение этому типу?», сохраняя вывод типа выражения.

---

## 1. Что делает `as`

`as` создаёт type assertion — утверждение типа.

```ts
const input = document.querySelector('#email') as HTMLInputElement;

console.log(input.value);
```

Без assertion результат `querySelector` имеет тип `Element | null`. Через `as HTMLInputElement` разработчик сообщает компилятору, что значение нужно считать `HTMLInputElement`.

### `as` не выполняет runtime-проверку

```ts
type User = {
  id: number;
  name: string;
};

const value: unknown = null;
const user = value as User;

console.log(user.name); // runtime-ошибка
```

TypeScript разрешил обращение к `name`, но `value` остался `null`. Assertion:

- не проверяет структуру;
- не создаёт объект;
- не преобразует данные;
- не добавляет свойства;
- удаляется при компиляции.

### `as` меняет статический тип выражения

```ts
type Status = 'pending' | 'success' | 'error';

const status = 'success' as Status;
// status: Status
```

Исходное значение было литералом `'success'`, но после assertion переменная рассматривается как весь union `Status`.

---

## 2. Что делает `satisfies`

Оператор `satisfies` появился в TypeScript 4.9. Он проверяет совместимость выражения с типом, сохраняя информацию, выведенную для самого выражения.

```ts
type Palette = Record<
  'red' | 'green' | 'blue',
  string | [number, number, number]
>;

const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
} satisfies Palette;
```

TypeScript проверит:

- наличие всех обязательных ключей;
- отсутствие опечаток в свежем объектном литерале;
- соответствие значений типу `string | [number, number, number]`.

При этом сохраняется знание о конкретных значениях:

```ts
palette.green.toUpperCase(); // допустимо: green — string
palette.red.at(0);           // допустимо: red — tuple/array
```

Если бы `palette` была аннотирована как `Palette`, свойства получили бы более широкий union `string | [number, number, number]` и перед специфической операцией понадобилось бы сужение.

---

## 3. Основное различие на одном примере

```ts
type RouteConfig = {
  path: string;
  method: 'GET' | 'POST';
};
```

### Через аннотацию

```ts
const route: RouteConfig = {
  path: '/users',
  method: 'GET',
};

// route.method: 'GET' | 'POST'
```

Аннотация задаёт тип переменной `RouteConfig`.

### Через `as`

```ts
const route = {
  path: '/users',
  method: 'GET',
} as RouteConfig;

// route рассматривается как RouteConfig
```

Assertion просит компилятор считать выражение `RouteConfig`.

### Через `satisfies`

```ts
const route = {
  path: '/users',
  method: 'GET',
} satisfies RouteConfig;

// route.method сохраняет более точный тип 'GET'
```

`satisfies` проверяет контракт, но не заменяет тип выражения на `RouteConfig`.

---

## 4. Сводная таблица

| Свойство | `as Type` | `satisfies Type` |
| --- | --- | --- |
| Назначение | Утверждает тип выражения | Проверяет соответствие типу |
| Меняет воспринимаемый тип выражения | Да | Обычно сохраняет выведенный тип |
| Проверяет совместимость | Ограниченно, допускает assertion при достаточном пересечении | Да |
| Сохраняет точные свойства и литералы | Часто нет | Да, насколько позволяет contextual typing |
| Может скрыть ошибку модели | Да | Обычно обнаруживает её |
| Выполняет runtime-валидацию | Нет | Нет |
| Преобразует значение | Нет | Нет |
| Существует в JavaScript | Нет | Нет |
| Подходит для проверки конфигурации | Рискованно | Да |
| Подходит, когда разработчик знает runtime-тип лучше компилятора | Да, осторожно | Не для этой задачи |

---

## 5. Проверка опечаток и лишних ключей

```ts
type Theme = {
  primary: string;
  secondary: string;
};
```

### Assertion может скрыть проблему

```ts
const theme = {
  primary: '#000',
  seconday: '#fff', // опечатка
} as Theme;
```

В зависимости от совместимости TypeScript может сообщить ошибку, но assertions в целом предназначены не для строгой проверки объектных конфигураций и нередко используются для подавления диагностики.

Особенно опасен двойной assertion:

```ts
const theme = {
  primary: '#000',
  seconday: '#fff',
} as unknown as Theme;
```

Он заставит компилятор принять почти любое значение.

### `satisfies` обнаруживает ошибку

```ts
const theme = {
  primary: '#000',
  seconday: '#fff', // ошибка: неизвестное свойство
} satisfies Theme;
```

Также будет обнаружено отсутствие `secondary`.

Для объектных конфигураций `satisfies` обычно безопаснее, потому что его задача — именно проверить контракт.

---

## 6. Сохранение ключей объекта

```ts
type Route = {
  path: string;
};
```

### Аннотация через `Record`

```ts
const routes: Record<string, Route> = {
  users: { path: '/users' },
  orders: { path: '/orders' },
};

type RouteName = keyof typeof routes;
// string
```

Из-за широкой аннотации конкретные ключи потеряны: известен любой `string`.

### Проверка через `satisfies`

```ts
const routes = {
  users: { path: '/users' },
  orders: { path: '/orders' },
} satisfies Record<string, Route>;

type RouteName = keyof typeof routes;
// 'users' | 'orders'
```

Контракт значений проверен, а точный union ключей сохранён.

Это один из самых полезных реальных сценариев `satisfies`.

---

## 7. Сохранение разных типов свойств

```ts
type Settings = Record<
  string,
  string | number | boolean
>;
```

### С аннотацией

```ts
const settings: Settings = {
  apiUrl: '/api',
  retries: 3,
  debug: true,
};

// settings.apiUrl: string | number | boolean
```

Перед вызовом строкового метода потребуется narrowing.

### С `satisfies`

```ts
const settings = {
  apiUrl: '/api',
  retries: 3,
  debug: true,
} satisfies Settings;

settings.apiUrl.toUpperCase(); // допустимо
settings.retries.toFixed(0);   // допустимо
```

Каждое свойство сохраняет свой выведенный тип, но весь объект проверяется как `Settings`.

---

## 8. `satisfies` не означает полную неизменность типа

Фраза «`satisfies` вообще никак не влияет на вывод типов» является слишком грубой. Правильнее сказать:

> `satisfies` проверяет выражение и сохраняет его результирующий тип вместо замены на целевой тип, но целевой тип участвует в contextual typing выражения.

Например, литералы объектов без `as const` обычно расширяются:

```ts
const config = {
  port: 3000,
} satisfies { port: number };

// config.port обычно number, а не литерал 3000
config.port = 4000; // допустимо
```

Для сохранения литералов и `readonly` используется `as const`:

```ts
const config = {
  port: 3000,
} as const satisfies { port: number };

// config.port: 3000 и readonly
```

---

## 9. `as const` и `satisfies`

Эти конструкции решают разные задачи и хорошо работают вместе.

### `as const`

`as const`:

- сохраняет литеральные типы;
- делает свойства объектного литерала `readonly`;
- превращает массив в readonly tuple.

```ts
const roles = ['admin', 'editor', 'viewer'] as const;

type Role = (typeof roles)[number];
// 'admin' | 'editor' | 'viewer'
```

### `as const satisfies`

```ts
type Route = {
  path: string;
  auth: boolean;
};

const routes = {
  users: {
    path: '/users',
    auth: true,
  },
  login: {
    path: '/login',
    auth: false,
  },
} as const satisfies Record<string, Route>;
```

Результат:

- структура проверена как `Record<string, Route>`;
- ключи сохранены как `'users' | 'login'`;
- пути сохранены как литералы `'/users'` и `'/login'`;
- свойства стали `readonly`.

Порядок важен:

```ts
expression as const satisfies Type
```

Сначала применяется const assertion к литералу, затем результат проверяется на соответствие типу.

---

## 10. `satisfies` и аннотация типа

Сравнение нужно проводить не только с `as`, но и с обычной аннотацией.

```ts
type Config = {
  mode: 'development' | 'production';
  port: number;
};
```

### Аннотация

```ts
const config: Config = {
  mode: 'development',
  port: 3000,
};
```

Преимущества:

- значение проверяется;
- переменная имеет ровно публичный тип `Config`;
- детали реализации намеренно скрываются.

Недостаток:

- более точный тип выражения может расшириться до `Config`.

### `satisfies`

```ts
const config = {
  mode: 'development',
  port: 3000,
} satisfies Config;
```

Преимущества:

- значение проверяется;
- более точный тип выражения сохраняется.

Выбор:

- если переменная должна иметь ровно заданный публичный тип — используйте аннотацию;
- если нужно проверить контракт и сохранить конкретные ключи или типы свойств — используйте `satisfies`.

---

## 11. Когда `as` оправдан

`as` не является запрещённым оператором. Он нужен, когда разработчик действительно знает о runtime-значении больше, чем компилятор.

### DOM API

```ts
const input = document.querySelector<HTMLInputElement>('#email');

if (input) {
  console.log(input.value);
}
```

Generic-вариант часто лучше assertion. Но если API не позволяет передать тип и наличие элемента гарантируется внешним контрактом:

```ts
const input = document.querySelector('#email') as HTMLInputElement;
```

Нужно помнить, что элемент всё ещё может отсутствовать в runtime.

### После собственной runtime-валидации

Иногда библиотека-валидатор гарантирует структуру, но её типы не отражают результат достаточно точно. Тогда assertion может быть обоснованным на границе проверенного кода.

### Работа с неполной информацией библиотеки

Assertion может применяться при неточных или устаревших типах внешней библиотеки. Желательно локализовать его в одном месте и добавить пояснение.

### Сужение, которое TypeScript не способен вывести

Иногда сложный инвариант уже доказан логикой программы, но control flow analysis его не понимает. Перед `as` стоит рассмотреть:

- type guard;
- assertion function;
- discriminated union;
- изменение модели данных;
- generic API.

---

## 12. Когда выбирать `satisfies`

`satisfies` особенно полезен для:

- конфигурационных объектов;
- словарей и реестров;
- таблиц маршрутов;
- объектов темизации;
- mapping между событиями и обработчиками;
- проверки полноты набора ключей;
- сохранения конкретного `keyof`;
- сохранения различий между типами свойств;
- данных с `as const`.

```ts
type EventName = 'login' | 'logout';

type EventHandler = (payload: unknown) => void;

const handlers = {
  login: payload => console.log('login', payload),
  logout: payload => console.log('logout', payload),
} satisfies Record<EventName, EventHandler>;
```

Если добавить лишний ключ или забыть обязательный, TypeScript сообщит об ошибке.

---

## 13. `satisfies` не является runtime-валидацией

```ts
type User = {
  id: number;
  name: string;
};

const user = {
  id: 1,
  name: 'Alex',
} satisfies User;
```

Проверяется выражение, уже известное TypeScript во время компиляции. Но данные API приходят в runtime:

```ts
const response = await fetch('/api/user');
const data: unknown = await response.json();
```

Нельзя безопасно решить проблему так:

```ts
// Не превращает данные в проверенного User
const user = data as User;
```

И `satisfies` не заменяет валидатор внешних данных. Нужен type guard или runtime-схема:

```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'number' &&
    'name' in value &&
    typeof value.name === 'string'
  );
}
```

---

## 14. Ограничения assertions

TypeScript не разрешает совершенно произвольный assertion напрямую, если типы недостаточно пересекаются:

```ts
const value = 'hello' as number;
// ошибка: типы недостаточно пересекаются
```

Но ограничение можно обойти через `unknown`:

```ts
const value = 'hello' as unknown as number;
```

Это называется double assertion. Код компилируется, но строка не превращается в число.

Двойной assertion — сильный сигнал риска. Он допустим только в редких интеграционных сценариях с внешними гарантиями и должен быть локализован.

---

## 15. Generic helper vs `satisfies`

До появления `satisfies` для проверки и сохранения вывода типов часто создавали identity-функции:

```ts
type Route = {
  path: string;
};

function defineRoutes<
  T extends Record<string, Route>,
>(routes: T): T {
  return routes;
}

const routes = defineRoutes({
  users: { path: '/users' },
  orders: { path: '/orders' },
});
```

Во многих простых случаях теперь достаточно:

```ts
const routes = {
  users: { path: '/users' },
  orders: { path: '/orders' },
} satisfies Record<string, Route>;
```

Но helper остаётся полезным, если нужна:

- runtime-обработка;
- зависимость типов между несколькими аргументами;
- нормализация данных;
- сложный generic inference;
- удобный публичный API библиотеки.

---

## 16. Частые ошибки

### Использование `as` для данных API

```ts
const user = (await response.json()) as User;
```

Это не проверяет ответ. Безопаснее получить `unknown` и валидировать.

### Ожидание runtime-проверки от `satisfies`

`satisfies` работает только при компиляции и исчезает из JavaScript.

### Ожидание, что `satisfies` присвоит переменной целевой тип

```ts
const config = expression satisfies Config;
```

`config` получает тип выражения, а не автоматически тип `Config`.

### Ожидание глубокого `readonly` от `satisfies`

`satisfies` ничего не делает неизменяемым. Для литеральной readonly-структуры нужен `as const` или отдельный readonly-тип.

### Использование `as const` вместо проверки контракта

`as const` сохраняет литералы, но не проверяет соответствие бизнес-модели. Для обеих задач:

```ts
const value = {
  // ...
} as const satisfies ExpectedType;
```

### Утверждение, что `satisfies` совсем не влияет на inference

Целевой тип участвует в contextual typing. Корректная формулировка: результат сохраняет тип выражения вместо замены на целевой тип.

---

## 17. Популярные вопросы на интервью

### В чём отличие `as` от `satisfies`?

`as` утверждает, каким типом считать выражение, и меняет его статический тип. `satisfies` проверяет совместимость выражения с типом, но сохраняет более точный тип выражения.

### Выполняет ли `as` преобразование данных?

Нет. `value as number` не превращает строку в число. Для преобразования нужен runtime-код, например `Number(value)`.

### Выполняет ли `satisfies` runtime-валидацию?

Нет. Оператор работает только на этапе компиляции и удаляется из JavaScript.

### Чем `satisfies` отличается от аннотации переменной?

Аннотация задаёт переменной указанный тип и может расширить конкретные свойства до него. `satisfies` проверяет контракт, сохраняя результирующий тип выражения.

### Может ли `satisfies` заменить `as`?

Не всегда. Они решают разные задачи. `satisfies` подходит для проверки известных выражений, а `as` нужен, когда разработчик обладает дополнительной информацией о runtime-значении, которую компилятор вывести не может.

### Что безопаснее для объекта конфигурации?

Обычно `satisfies`, потому что он проверяет обязательные поля, типы значений и опечатки, не теряя точные ключи.

### Для чего используется `as const satisfies`?

`as const` сохраняет литеральные типы и делает структуру readonly, а `satisfies` проверяет её соответствие ожидаемому контракту.

### Почему `as unknown as Type` опасен?

Он позволяет обойти проверку совместимости почти для любых типов. Runtime-значение при этом не меняется.

### Сохраняет ли `satisfies` литерал `3000`?

Не всегда сам по себе: литералы изменяемых свойств могут расширяться до `number`. Для гарантированного сохранения литеральных значений обычно используют `as const satisfies`.

### Можно ли использовать `satisfies` с union?

Да:

```ts
type Action =
  | { type: 'add'; value: number }
  | { type: 'remove'; id: string };

const action = {
  type: 'add',
  value: 10,
} satisfies Action;
```

Объект проверяется как член union, а его конкретный вариант сохраняется.

---

## 18. Практические задачи с ответами

### Задача 1: какой тип будет у `method`?

```ts
type RequestConfig = {
  method: 'GET' | 'POST';
};

const first = {
  method: 'GET',
} as RequestConfig;

const second = {
  method: 'GET',
} satisfies RequestConfig;
```

**Ответ:**

- `first.method` рассматривается как `'GET' | 'POST'`;
- `second.method` сохраняет более конкретный тип `'GET'`.

### Задача 2: найдёт ли TypeScript опечатку?

```ts
type Config = {
  timeout: number;
};

const config = {
  timout: 5000,
} satisfies Config;
```

**Ответ:** да. Свойство `timout` неизвестно, а обязательное `timeout` отсутствует.

### Задача 3: почему теряются ключи?

```ts
type Route = { path: string };

const routes: Record<string, Route> = {
  home: { path: '/' },
  users: { path: '/users' },
};

type RouteName = keyof typeof routes;
```

**Ответ:** аннотация `Record<string, Route>` расширила ключи до `string`. Для сохранения конкретных ключей:

```ts
const routes = {
  home: { path: '/' },
  users: { path: '/users' },
} satisfies Record<string, Route>;

type RouteName = keyof typeof routes;
// 'home' | 'users'
```

### Задача 4: безопасен ли код?

```ts
const value = JSON.parse(text) as User;
```

**Ответ:** нет. `JSON.parse` возвращает runtime-данные, а `as User` не проверяет их. Нужна валидация.

### Задача 5: сохранить литералы и проверить контракт

```ts
type Environment = {
  mode: 'development' | 'production';
  port: number;
};
```

**Решение:**

```ts
const environment = {
  mode: 'development',
  port: 3000,
} as const satisfies Environment;
```

### Задача 6: что попадёт в JavaScript?

```ts
const config = {
  port: 3000,
} satisfies { port: number };
```

**Ответ:** обычный объект. Оператор `satisfies` удаляется:

```js
const config = {
  port: 3000,
};
```

---

## 19. Развёрнутый ответ для собеседования

> `as` — это type assertion. Он сообщает TypeScript, каким типом считать выражение, и может заменить выведенный тип на указанный. При этом никакой runtime-проверки или преобразования не происходит, поэтому необоснованный `as`, особенно для данных API, способен скрыть ошибку.
>
> `satisfies` проверяет, что выражение совместимо с заданным типом, но сохраняет результирующий тип самого выражения. Благодаря этому можно проверить объект конфигурации и одновременно не потерять его конкретные ключи, литеральные варианты и различия между типами свойств. Например, при проверке объекта через `satisfies Record<string, Route>` `keyof` останется union реальных ключей, а не просто `string`.
>
> `satisfies` не заменяет `as` во всех случаях и не выполняет runtime-валидацию. `as` применяют, когда разработчик действительно знает о runtime-значении больше компилятора. `satisfies` применяют, когда компилятор должен проверить известное выражение. Если дополнительно нужны литеральные readonly-типы, используют `as const satisfies Type`.

---

## 20. Чек-лист перед интервью

- Называть оператор правильно: `satisfies`.
- Объяснить, что `as` — утверждение, а `satisfies` — проверка.
- Помнить, что оба оператора отсутствуют в runtime.
- Показать, что `as` не преобразует данные.
- Объяснить, почему `as` опасен для ответа API.
- Сравнить `satisfies` с обычной аннотацией.
- Показать сохранение `keyof` через `satisfies`.
- Объяснить contextual typing и сохранение типа выражения.
- Показать `as const satisfies`.
- Назвать допустимые сценарии `as`.
- Объяснить риск `as unknown as Type`.
- Не называть `satisfies` runtime-валидатором.
