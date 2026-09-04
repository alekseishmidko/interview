# `type` vs `interface` в TypeScript — подготовка к интервью

## Короткий ответ для собеседования

`type` и `interface` в TypeScript часто решают одну задачу — описывают форму объекта. Оба поддерживают свойства, методы, дженерики, расширение и могут использоваться классом через `implements`.

Главные различия:

- `type` универсальнее: может описывать примитивы, union, tuple, функции, mapped и conditional types;
- `interface` ориентирован на объектные контракты и поддерживает **declaration merging** — объявления с одинаковым именем объединяются;
- `interface` расширяется через `extends`, а типы обычно комбинируются через `&`;
- конфликты свойств при `extends` обычно дают понятную ошибку сразу, а пересечение способно создать поле типа `never`.

Практический выбор: `interface` удобно использовать для расширяемых публичных контрактов объектов и классов, а `type` — для union, tuple, функций и преобразований типов. Для обычной модели объекта оба варианта корректны, поэтому важно соблюдать соглашения проекта.

---

## 1. Что такое `type`

`type` создаёт псевдоним для любого допустимого типа TypeScript.

```ts
type User = {
  id: number;
  name: string;
};
```

Через `type` можно описать не только объект:

```ts
type ID = string | number;

type Status = 'idle' | 'loading' | 'success' | 'error';

type Point = [number, number];

type Handler = (event: Event) => void;
```

`type` не создаёт новое значение или конструктор в JavaScript. Это конструкция системы типов, которая исчезает после компиляции.

---

## 2. Что такое `interface`

`interface` описывает контракт объектной структуры.

```ts
interface User {
  id: number;
  name: string;
}
```

Интерфейс может содержать:

- обязательные и необязательные свойства;
- свойства `readonly`;
- методы;
- call signature;
- construct signature;
- индексные сигнатуры;
- дженерики.

```ts
interface UserService {
  readonly url: string;
  timeout?: number;
  getUser(id: number): Promise<User>;
}
```

Как и `type`, `interface` существует только на этапе проверки TypeScript и отсутствует в JavaScript runtime.

---

## 3. Описание объекта

Для простой объектной модели возможности почти одинаковы.

### Через `type`

```ts
type Product = {
  readonly id: number;
  title: string;
  description?: string;
  getPrice(currency: string): number;
};
```

### Через `interface`

```ts
interface Product {
  readonly id: number;
  title: string;
  description?: string;
  getPrice(currency: string): number;
}
```

При использовании этих моделей разницы практически нет:

```ts
const product: Product = {
  id: 1,
  title: 'Phone',
  getPrice: () => 1000,
};
```

TypeScript использует структурную типизацию: совместимость определяется структурой значения, а не тем, было ли оно явно объявлено через конкретный `type` или `interface`.

---

## 4. Расширение: `extends` и intersection

### Расширение интерфейса

```ts
interface User {
  id: number;
  name: string;
}

interface Admin extends User {
  permissions: string[];
}
```

`Admin` содержит все поля `User` и собственное поле `permissions`.

Интерфейс может расширять несколько интерфейсов:

```ts
interface WithTimestamps {
  createdAt: Date;
  updatedAt: Date;
}

interface Admin extends User, WithTimestamps {
  permissions: string[];
}
```

### Комбинирование типов через `&`

```ts
type User = {
  id: number;
  name: string;
};

type WithTimestamps = {
  createdAt: Date;
  updatedAt: Date;
};

type Admin = User &
  WithTimestamps & {
    permissions: string[];
  };
```

Intersection означает, что значение должно удовлетворять всем объединённым требованиям.

### Можно смешивать `type` и `interface`

Тип может пересекаться с интерфейсом:

```ts
interface User {
  id: number;
}

type Admin = User & {
  permissions: string[];
};
```

Интерфейс может расширить объектный type alias, если его члены статически известны:

```ts
type Named = {
  name: string;
};

interface User extends Named {
  id: number;
}
```

Но интерфейс не может напрямую расширить произвольный union:

```ts
type Entity =
  | { type: 'user'; userId: number }
  | { type: 'company'; companyId: number };

// interface AuditedEntity extends Entity {} // ошибка
```

Union не представляет одну объектную структуру со статически известными членами.

---

## 5. Разное поведение при конфликте свойств

Это важное различие для собеседования.

### Конфликт через `interface extends`

```ts
interface BaseEntity {
  id: string;
}

interface User extends BaseEntity {
  id: number; // ошибка
}
```

TypeScript сразу сообщает, что `User` некорректно расширяет `BaseEntity`.

### Конфликт через intersection

```ts
type BaseEntity = {
  id: string;
};

type User = BaseEntity & {
  id: number;
};
```

Псевдоним может быть создан, но тип свойства становится пересечением:

```ts
type UserId = User['id']; // string & number → never
```

Создать нормальное значение `User` невозможно, потому что `id` должен одновременно быть строкой и числом.

Вывод:

> `extends` проверяет совместимость переопределяемых членов, а `&` механически пересекает требования и при конфликте может привести к `never`.

---

## 6. Declaration merging

Интерфейсы с одинаковым именем в одной области могут объединяться.

```ts
interface AppConfig {
  apiUrl: string;
}

interface AppConfig {
  timeout: number;
}

const config: AppConfig = {
  apiUrl: '/api',
  timeout: 5000,
};
```

Итоговый `AppConfig` содержит оба свойства.

Для `type` повторное объявление запрещено:

```ts
type AppConfig = {
  apiUrl: string;
};

// Ошибка: Duplicate identifier 'AppConfig'
type AppConfig = {
  timeout: number;
};
```

### Где declaration merging полезен

Он применяется для расширения существующих деклараций библиотек и глобальных объектов:

```ts
declare global {
  interface Window {
    analytics: {
      track(event: string): void;
    };
  }
}
```

После этого TypeScript знает о `window.analytics`.

Declaration merging также используется при module augmentation:

```ts
declare module 'express-session' {
  interface SessionData {
    userId: string;
  }
}
```

### Возможный недостаток

Открытость интерфейса означает, что его контракт может быть дополнен в другом файле. Для расширяемых API это преимущество, но для закрытой доменной модели иногда нежелательно. `type` нельзя случайно повторно открыть.

---

## 7. Union types

`type` может напрямую описать объединение:

```ts
type ID = string | number;

type Status = 'pending' | 'success' | 'error';
```

Интерфейс напрямую union не описывает:

```ts
// Некорректный синтаксис
// interface ID = string | number;
```

Особенно полезен discriminated union:

```ts
type RequestState =
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; error: Error };
```

```ts
function render(state: RequestState) {
  switch (state.status) {
    case 'loading':
      return 'Loading';
    case 'success':
      return state.data.join(', ');
    case 'error':
      return state.error.message;
  }
}
```

Для описания вариантов состояния `type` обычно является естественным выбором.

---

## 8. Примитивы и литеральные типы

Только `type` может напрямую именовать примитив или набор литералов:

```ts
type UserId = string;

type Theme = 'light' | 'dark';

type NullableString = string | null;
```

`interface` предназначен для объектных форм и не может быть псевдонимом примитива.

---

## 9. Tuple

Для кортежей обычно используют `type`:

```ts
type Coordinates = [latitude: number, longitude: number];

type ApiResult<T> = [data: T, statusCode: number];
```

```ts
const point: Coordinates = [16.0544, 108.2022];
```

Технически интерфейс может расширять некоторые массивоподобные структуры, но для tuple это громоздко и хуже передаёт намерение. В практическом коде tuple описывают через `type`.

---

## 10. Функции и callable-объекты

Обычную функцию можно описать обоими способами.

### Через `type`

```ts
type Formatter = (value: string) => string;
```

### Через `interface`

```ts
interface Formatter {
  (value: string): string;
}
```

Для простой сигнатуры `type` обычно короче. Интерфейс удобен, если у вызываемого объекта есть дополнительные свойства:

```ts
interface Formatter {
  (value: string): string;
  locale: string;
  reset(): void;
}
```

То же самое возможно и через `type`:

```ts
type Formatter = {
  (value: string): string;
  locale: string;
  reset(): void;
};
```

---

## 11. Дженерики

И `type`, и `interface` поддерживают параметры типа.

### Generic type

```ts
type ApiResponse<T> = {
  data: T;
  status: number;
};
```

### Generic interface

```ts
interface ApiResponse<T> {
  data: T;
  status: number;
}
```

Использование одинаковое:

```ts
type UserResponse = ApiResponse<User>;
```

Оба варианта поддерживают ограничения и значения по умолчанию:

```ts
interface Repository<T extends { id: string | number }> {
  findById(id: T['id']): Promise<T | null>;
}

type Result<T = unknown> = {
  value: T;
};
```

Наличие дженерика само по себе не является причиной выбирать только `type` или только `interface`.

---

## 12. Mapped и conditional types

Для преобразования типов нужен `type`.

### Mapped type

```ts
type Nullable<T> = {
  [Key in keyof T]: T[Key] | null;
};
```

### Conditional type

```ts
type UnwrapPromise<T> =
  T extends Promise<infer Value>
    ? Value
    : T;
```

```ts
type Result = UnwrapPromise<Promise<User>>; // User
```

Интерфейс не может напрямую быть mapped или conditional type. Но он может использовать результат такого type alias:

```ts
type NullableUser = Nullable<User>;

interface Response {
  user: NullableUser;
}
```

---

## 13. Классы и `implements`

Класс может реализовать и интерфейс, и объектный type alias.

### С интерфейсом

```ts
interface Named {
  name: string;
  getName(): string;
}

class User implements Named {
  constructor(public name: string) {}

  getName() {
    return this.name;
  }
}
```

### С type alias

```ts
type Named = {
  name: string;
  getName(): string;
};

class User implements Named {
  constructor(public name: string) {}

  getName() {
    return this.name;
  }
}
```

Класс не может реализовать произвольный union:

```ts
type Result =
  | { success: true; data: string }
  | { success: false; error: string };

// class ApiResult implements Result {} // ошибка
```

`implements` требует объектный тип со статически известными членами.

Важно: `implements` только проверяет публичную структуру класса. Он не добавляет методы и не создаёт runtime-наследование. Для наследования реализации используется `extends` класса.

---

## 14. Индексные сигнатуры

Индексную сигнатуру можно описать обоими способами.

```ts
interface Scores {
  [userId: string]: number;
}
```

```ts
type Scores = {
  [userId: string]: number;
};
```

Для ограниченного набора ключей удобнее mapped type или `Record`:

```ts
type Role = 'admin' | 'editor' | 'viewer';
type Permissions = Record<Role, string[]>;
```

Mapped-синтаксис непосредственно внутри `interface` использовать нельзя.

---

## 15. Рекурсивные типы

Современный TypeScript позволяет создавать рекурсивные структуры обоими способами.

### Через `interface`

```ts
interface TreeNode {
  value: string;
  children: TreeNode[];
}
```

### Через `type`

```ts
type TreeNode = {
  value: string;
  children: TreeNode[];
};
```

Для union-рекурсии нужен `type`:

```ts
type Json =
  | string
  | number
  | boolean
  | null
  | Json[]
  | { [key: string]: Json };
```

---

## 16. Перегрузки и объединение сигнатур

Интерфейс может содержать несколько call signatures:

```ts
interface Parser {
  (value: string): object;
  (value: Uint8Array): object;
}
```

Type alias тоже может описать перегруженный вызываемый объект:

```ts
type Parser = {
  (value: string): object;
  (value: Uint8Array): object;
};
```

Но declaration merging делает интерфейс удобным, когда сигнатуры дополняются в разных декларациях или при расширении библиотечных типов.

---

## 17. Utility types

Utility types возвращают новые типы, поэтому результат обычно сохраняют через `type`:

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

type CreateUserDto = Omit<User, 'id'>;
type UpdateUserDto = Partial<Pick<User, 'name' | 'email'>>;
type ReadonlyUser = Readonly<User>;
```

Исходная модель может быть интерфейсом, а производные типы — type aliases. Их необязательно противопоставлять: в одном проекте они хорошо работают вместе.

---

## 18. Excess property checks

Для объектных литералов `type` и `interface` ведут себя похожим образом.

```ts
interface User {
  id: number;
}

const user: User = {
  id: 1,
  name: 'Alex', // ошибка: лишнее свойство
};
```

Аналогичная проверка будет с объектным type alias.

Но если значение сначала сохранить в переменной, совместимость проверяется структурно:

```ts
const value = {
  id: 1,
  name: 'Alex',
};

const user: User = value; // допустимо
```

Это не различие `type` и `interface`, а общее поведение TypeScript при проверке свежих объектных литералов.

---

## 19. Что происходит в runtime

И `type`, и `interface` удаляются при компиляции:

```ts
type UserType = {
  id: number;
};

interface UserInterface {
  id: number;
}
```

В скомпилированном JavaScript этих деклараций не будет.

Поэтому нельзя написать:

```ts
value instanceof UserType;
value instanceof UserInterface;
```

`instanceof` требует реальное значение — например класс:

```ts
class User {
  constructor(public id: number) {}
}

value instanceof User;
```

Для проверки данных API применяют runtime-валидацию или type guard:

```ts
interface User {
  id: number;
  name: string;
}

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

Выбор между `type` и `interface` не обеспечивает runtime-безопасность.

---

## 20. Сводная таблица

| Возможность | `type` | `interface` |
| --- | --- | --- |
| Описать объект | Да | Да |
| Необязательные и `readonly` свойства | Да | Да |
| Методы и call signatures | Да | Да |
| Дженерики | Да | Да |
| Расширение объектной модели | Через `&` | Через `extends` |
| Наследование нескольких моделей | Через `&` | Несколько `extends` |
| Реализация классом | Да, для объектного типа | Да |
| Union | Да | Нет напрямую |
| Intersection | Да | Не напрямую, используется `extends` |
| Примитивный alias | Да | Нет |
| Литеральный тип | Да | Нет |
| Tuple | Да | Практически используют `type` |
| Mapped type | Да | Нет напрямую |
| Conditional type | Да | Нет |
| Declaration merging | Нет | Да |
| Module augmentation | Не переоткрывается | Поддерживается |
| Существует в runtime | Нет | Нет |

---

## 21. Когда выбирать `interface`

`interface` хорошо подходит, когда:

- описывается объектный контракт;
- контракт предполагается расширять через `extends`;
- тип реализуется классом;
- создаётся публичный API библиотеки;
- требуется declaration merging или module augmentation;
- команда договорилась использовать интерфейсы для доменных моделей.

```ts
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}
```

Однако эти сценарии не означают, что объектный `type` был бы технически неправильным.

---

## 22. Когда выбирать `type`

`type` хорошо подходит, когда нужен:

- union или discriminated union;
- intersection;
- tuple;
- псевдоним примитива или литерала;
- тип функции в компактной форме;
- mapped или conditional type;
- результат `Pick`, `Omit`, `Partial`, `ReturnType` и других преобразований;
- закрытый псевдоним, который нельзя дополнить declaration merging.

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

---

## 23. Практическая стратегия в проекте

Универсального правила «всегда использовать только одно» нет. Удобная стратегия:

1. Для union, tuple, функций и type-level преобразований использовать `type`.
2. Для открытых объектных контрактов и расширяемых деклараций использовать `interface`.
3. Для обычных внутренних DTO и моделей выбрать единое командное соглашение.
4. Не переписывать существующий код только ради замены одного синтаксиса другим без практической причины.
5. Оценивать контракт: должен ли он быть открыт для расширения или оставаться закрытым.

Пример совместного использования:

```ts
interface User {
  id: string;
  name: string;
  email: string;
}

type CreateUserDto = Omit<User, 'id'>;

type UpdateUserDto = Partial<CreateUserDto>;

type UserLoadingState =
  | { status: 'loading' }
  | { status: 'success'; user: User }
  | { status: 'error'; message: string };
```

---

## 24. Частые ошибки и заблуждения

### «`interface` может описывать объект, а `type` не может»

Неверно. Оба могут описывать объектную структуру.

### «`type` нельзя расширять»

Неверно. Type aliases комбинируются через intersection `&`. Кроме того, объектный type alias может быть основой для `interface extends`.

### «Класс может реализовать только `interface`»

Неверно. Класс может реализовать объектный type alias со статически известными членами.

### «`interface` существует в JavaScript»

Неверно. И `interface`, и `type` удаляются при компиляции.

### «`type` всегда лучше, потому что умеет больше»

Не обязательно. Открытое расширение, declaration merging и читаемое наследование объектных контрактов делают `interface` полезным инструментом.

### «`interface` всегда быстрее компилируется»

Это слишком общее утверждение. Реальная производительность зависит от структуры проекта и сложности типов. Не стоит выбирать синтаксис только по этому правилу без измерений.

### «Intersection полностью эквивалентен `extends`»

Нет. При конфликтующих свойствах они ведут себя по-разному: `extends` сообщает о несовместимости, а `&` способен создать `never`.

---

## 25. Популярные вопросы на интервью

### В чём основное отличие `type` от `interface`?

`interface` описывает расширяемый объектный контракт и поддерживает declaration merging. `type` является псевдонимом любого типа и позволяет напрямую создавать union, tuple, mapped и conditional types.

### Что выбрать для обычного объекта?

Оба варианта корректны. Выбор зависит от необходимости declaration merging, способа расширения и соглашений проекта.

### Можно ли расширить `type`?

Да, через intersection:

```ts
type Admin = User & { permissions: string[] };
```

### Может ли `interface` расширять `type`?

Да, если type alias описывает объектную структуру со статически известными членами. Произвольный union интерфейс расширить не может.

### Может ли класс реализовать `type`?

Да, если type alias является объектным типом со статически известными членами. Union класс реализовать не может.

### Что такое declaration merging?

Это автоматическое объединение совместимых объявлений интерфейса с одинаковым именем. У `type` такой возможности нет.

### Чем `extends` отличается от `&`?

Оба механизма комбинируют требования, но `extends` проверяет совместимость переопределяемых свойств. Intersection пересекает типы и при конфликте может получить `never`.

### Что лучше для DTO?

Технически подходят оба. Часто базовую объектную модель описывают интерфейсом, а производные DTO — через `type` и utility types: `Pick`, `Omit`, `Partial`.

### Что лучше для React props?

Оба варианта работают одинаково хорошо. `type` удобен, если props образуют union вариантов, а `interface` — если контракт предполагается расширять. На практике следует соблюдать соглашение команды.

---

## 26. Задачи с разбором

### Задача 1: какой вариант нужен для состояния запроса?

```ts
type RequestState<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

**Ответ:** `type`, потому что требуется discriminated union.

### Задача 2: что произойдёт при повторном объявлении?

```ts
interface Config {
  apiUrl: string;
}

interface Config {
  timeout: number;
}
```

**Ответ:** объявления объединятся. Итоговый `Config` потребует `apiUrl` и `timeout`.

### Задача 3: почему поле стало `never`?

```ts
type A = { id: string };
type B = { id: number };
type C = A & B;
```

**Ответ:** `C['id']` должен одновременно быть `string` и `number`. Такие типы несовместимы, поэтому результат — `never`.

### Задача 4: можно ли реализовать тип классом?

```ts
type Serializable = {
  serialize(): string;
};

class User implements Serializable {
  serialize() {
    return '{}';
  }
}
```

**Ответ:** да. Type alias описывает объектный контракт со статически известным методом.

### Задача 5: чем лучше описать обработчик?

```ts
type ClickHandler = (event: MouseEvent) => void;
```

**Ответ:** оба варианта возможны, но `type` короче и естественнее для простой функции.

---

## 27. Развёрнутый ответ для собеседования

> `type` и `interface` могут описывать объектную структуру, поддерживают свойства, методы, дженерики и могут использоваться через `implements`. TypeScript структурно сравнивает такие модели, поэтому для обычного объекта они во многом взаимозаменяемы.
>
> Главное преимущество `type` — универсальность. Он может описывать union, intersection, tuple, функцию, литеральный или примитивный тип, а также mapped и conditional types. `interface` ориентирован на объектные контракты, расширяется через `extends` и поддерживает declaration merging, что полезно для публичных API и module augmentation.
>
> Ещё одно различие проявляется при конфликте свойств: несовместимое расширение интерфейса через `extends` сразу вызывает ошибку, а intersection может создать свойство типа `never`. На практике я использую `type` для union и преобразований типов, `interface` — для открытых объектных контрактов, а для обычных моделей следую соглашениям проекта.

---

## 28. Чек-лист перед интервью

- Уметь описать одинаковый объект через `type` и `interface`.
- Объяснить структурную типизацию TypeScript.
- Показать расширение через `extends` и `&`.
- Рассказать о конфликте свойств и возможном `never`.
- Объяснить declaration merging.
- Назвать сценарии module augmentation.
- Знать, что union и tuple напрямую создаются через `type`.
- Уметь показать generic `type` и generic `interface`.
- Объяснить использование обоих вариантов с `implements`.
- Помнить, что оба отсутствуют в runtime.
- Аргументировать выбор без категоричного «всегда только `type`» или «всегда только `interface`».
