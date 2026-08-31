# `type` в TypeScript — подготовка к интервью

## Краткий ответ для собеседования

`type` в TypeScript — ключевое слово для создания **псевдонима типа** (`type alias`). Оно позволяет дать имя типу объекта, примитиву, объединению, пересечению, кортежу, функции или результату преобразования других типов.

```ts
type User = {
  id: number;
  name: string;
  isAdmin?: boolean;
};

const user: User = {
  id: 1,
  name: 'Alex',
};
```

`type` существует только во время статической проверки TypeScript. После компиляции псевдоним исчезает и не создаёт значение в JavaScript runtime.

---

## 1. Что такое тип в TypeScript

Тип описывает множество допустимых значений и операций над ними. Например:

```ts
let count: number = 10;
let title: string = 'TypeScript';
```

Переменная `count` может содержать числа, а `title` — строки. TypeScript проверяет эти ограничения до выполнения программы.

Ключевое слово `type` позволяет сохранить описание типа под собственным именем:

```ts
type ID = string | number;

let userId: ID = 10;
userId = 'user-10';
```

`ID` — не новый runtime-тип и не новый конструктор. Это другое имя для типа `string | number`.

---

## 2. Тип объекта

```ts
type Product = {
  id: number;
  title: string;
  price: number;
  description?: string;
  readonly createdAt: Date;
};
```

Здесь:

- `id`, `title` и `price` — обязательные свойства;
- `description?` — необязательное свойство;
- `readonly createdAt` запрещено переназначать через типизированную ссылку.

```ts
const product: Product = {
  id: 1,
  title: 'Phone',
  price: 1000,
  createdAt: new Date(),
};

product.price = 900;             // допустимо
product.createdAt = new Date();  // ошибка TypeScript
```

`readonly` является ограничением системы типов. Само по себе оно не делает объект неизменяемым в JavaScript runtime.

---

## 3. Union type — объединение

Объединение означает: значение может соответствовать одному из нескольких типов.

```ts
type ID = string | number;
type RequestStatus = 'idle' | 'loading' | 'success' | 'error';
```

Строковые литералы позволяют ограничить набор значений:

```ts
let status: RequestStatus = 'loading';
status = 'success'; // допустимо
status = 'done';    // ошибка
```

При работе с union TypeScript разрешает только операции, безопасные для всех его вариантов. Перед специфической операцией тип нужно сузить:

```ts
function printId(id: ID) {
  if (typeof id === 'string') {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(0));
  }
}
```

---

## 4. Discriminated union

Дискриминируемое объединение описывает несколько состояний объекта через общее литеральное поле.

```ts
type RequestState =
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; error: Error };

function render(state: RequestState) {
  switch (state.status) {
    case 'loading':
      return 'Loading...';

    case 'success':
      return state.data.join(', ');

    case 'error':
      return state.error.message;
  }
}
```

Поле `status` является дискриминатором. После его проверки TypeScript понимает точный вариант объекта.

Такой подход безопаснее типа с большим количеством необязательных полей:

```ts
type WeakState = {
  status: string;
  data?: string[];
  error?: Error;
};
```

`WeakState` допускает невозможные состояния, например одновременное наличие `data` и `error`.

---

## 5. Intersection type — пересечение

Пересечение объединяет требования нескольких типов. Значение должно соответствовать им всем.

```ts
type User = {
  id: number;
  name: string;
};

type WithPermissions = {
  permissions: string[];
};

type Admin = User & WithPermissions;
```

```ts
const admin: Admin = {
  id: 1,
  name: 'Alex',
  permissions: ['users:read'],
};
```

Если пересекаются несовместимые свойства, может получиться `never`:

```ts
type A = { value: string };
type B = { value: number };
type Impossible = A & B;

// value должен одновременно быть string и number,
// поэтому его тип фактически становится never.
```

---

## 6. Тип функции

```ts
type Calculator = (left: number, right: number) => number;

const sum: Calculator = (left, right) => left + right;
```

Можно описать callback:

```ts
type SuccessHandler = (data: string[]) => void;

function loadData(onSuccess: SuccessHandler) {
  onSuccess(['one', 'two']);
}
```

Функция, возвращающая `void`, может технически вернуть значение — вызывающий код просто не должен использовать результат через этот тип. `void` не полностью равнозначен утверждению «функция никогда ничего не возвращает».

---

## 7. Массивы и кортежи

```ts
type Users = User[];
type Coordinates = [number, number];
type ApiResult = [data: string[], statusCode: number];
```

Массив содержит произвольное количество элементов одного типа. Кортеж задаёт тип и позицию элементов:

```ts
const point: Coordinates = [10, 20];
const result: ApiResult = [['a', 'b'], 200];
```

Для неизменяемого кортежа:

```ts
type ReadonlyPoint = readonly [number, number];
```

---

## 8. Индексная сигнатура и `Record`

Если заранее неизвестны имена ключей:

```ts
type Scores = {
  [userId: string]: number;
};

const scores: Scores = {
  alex: 10,
  max: 20,
};
```

Эквивалент через встроенный utility type:

```ts
type Scores = Record<string, number>;
```

Для ограниченного набора ключей `Record` особенно удобен:

```ts
type Role = 'admin' | 'editor' | 'viewer';
type Permissions = Record<Role, string[]>;
```

Теперь объект должен содержать все три ключа.

---

## 9. Generic type — обобщённый тип

Дженерик позволяет создавать типы, работающие с разными данными, сохраняя информацию о конкретном типе.

```ts
type ApiResponse<T> = {
  data: T;
  status: number;
  error?: string;
};

type UserResponse = ApiResponse<User>;
type UsersResponse = ApiResponse<User[]>;
```

Несколько параметров типа:

```ts
type Pair<Key, Value> = {
  key: Key;
  value: Value;
};

type UserPair = Pair<number, User>;
```

Ограничение параметра:

```ts
type Entity = { id: string | number };

type EntityResponse<T extends Entity> = {
  item: T;
};
```

Значение по умолчанию:

```ts
type Result<T = unknown> = {
  value: T;
};
```

---

## 10. `keyof`, `typeof` и indexed access types

### `keyof`

`keyof` создаёт union ключей типа:

```ts
type User = {
  id: number;
  name: string;
};

type UserKey = keyof User; // 'id' | 'name'
```

### TypeScript `typeof`

В позиции типа `typeof` получает тип существующего JavaScript-значения:

```ts
const config = {
  apiUrl: '/api',
  retries: 3,
};

type Config = typeof config;
```

Это не то же самое, что runtime-оператор `typeof value`, который возвращает строку вроде `'string'` или `'object'`.

### Indexed access type

```ts
type UserId = User['id'];       // number
type UserValue = User[keyof User]; // number | string
```

Эти операторы позволяют выводить типы из существующих моделей и не дублировать определения.

---

## 11. Utility types

TypeScript предоставляет готовые преобразования типов.

```ts
type User = {
  id: number;
  name: string;
  email: string;
};
```

### `Partial<T>`

Делает все свойства необязательными:

```ts
type UserPatch = Partial<User>;
```

### `Required<T>`

Делает все свойства обязательными:

```ts
type RequiredUser = Required<User>;
```

### `Readonly<T>`

Делает свойства доступными только для чтения:

```ts
type ReadonlyUser = Readonly<User>;
```

### `Pick<T, K>`

Выбирает указанные свойства:

```ts
type UserPreview = Pick<User, 'id' | 'name'>;
```

### `Omit<T, K>`

Исключает указанные свойства:

```ts
type CreateUserDto = Omit<User, 'id'>;
```

### `Exclude<T, U>` и `Extract<T, U>`

```ts
type Status = 'idle' | 'loading' | 'success' | 'error';

type FinishedStatus = Exclude<Status, 'idle' | 'loading'>;
// 'success' | 'error'

type ErrorStatus = Extract<Status, 'error' | 'cancelled'>;
// 'error'
```

### `ReturnType<T>` и `Parameters<T>`

```ts
function createUser() {
  return { id: 1, name: 'Alex' };
}

type CreatedUser = ReturnType<typeof createUser>;
type CreateUserParameters = Parameters<typeof createUser>;
```

Utility types обычно поверхностные. Например, `Partial<User>` не делает автоматически необязательными поля вложенных объектов.

---

## 12. Mapped type

Mapped type создаёт новый объектный тип, перебирая ключи другого типа.

```ts
type Nullable<T> = {
  [Key in keyof T]: T[Key] | null;
};

type NullableUser = Nullable<User>;
```

Можно менять модификаторы:

```ts
type Mutable<T> = {
  -readonly [Key in keyof T]: T[Key];
};

type AllRequired<T> = {
  [Key in keyof T]-?: T[Key];
};
```

Utility types вроде `Partial`, `Readonly`, `Pick` реализуются на основе подобных механизмов.

---

## 13. Conditional type и `infer`

Conditional type выбирает результат в зависимости от совместимости типов:

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

`infer` позволяет извлечь часть типа:

```ts
type UnwrapPromise<T> =
  T extends Promise<infer Value>
    ? Value
    : T;

type Result = UnwrapPromise<Promise<User>>; // User
```

Условный тип с «голым» параметром типа распределяется по union:

```ts
type ToArray<T> = T extends unknown ? T[] : never;
type Result = ToArray<string | number>;
// string[] | number[]
```

Это называется distributive conditional type.

---

## 14. `type` и `interface`

Оба способа подходят для описания структуры объекта:

```ts
type UserType = {
  id: number;
};

interface UserInterface {
  id: number;
}
```

Основные различия:

| Возможность | `type` | `interface` |
| --- | --- | --- |
| Описание объекта | Да | Да |
| Описание функции | Да | Да |
| Union | Да | Нет напрямую |
| Tuple | Да | Возможно, но `type` естественнее |
| Примитивный alias | Да | Нет |
| Conditional/mapped type | Да | Нет напрямую |
| Расширение | Пересечение `&` | `extends` |
| Declaration merging | Нет | Да |
| `implements` в классе | Да, если итог — объектный тип | Да |

### Расширение

```ts
type User = {
  id: number;
};

type Admin = User & {
  permissions: string[];
};
```

```ts
interface User {
  id: number;
}

interface Admin extends User {
  permissions: string[];
}
```

### Declaration merging

Интерфейсы с одинаковым именем объединяются:

```ts
interface WindowConfig {
  theme: string;
}

interface WindowConfig {
  language: string;
}
```

Итоговый `WindowConfig` содержит оба свойства. Повторно объявить `type` с тем же именем в одной области нельзя.

### Что выбрать

Практическое правило:

- `interface` удобно использовать для публичных расширяемых контрактов объектов и классов;
- `type` — для union, tuple, функций и преобразований типов;
- для обычной модели объекта оба варианта корректны;
- важнее придерживаться единого соглашения проекта.

---

## 15. `type` отсутствует в runtime

```ts
type User = {
  id: number;
};
```

После компиляции определения `User` в JavaScript не будет. Поэтому такой код невозможен:

```ts
value instanceof User; // ошибка: User существует только как тип
```

`instanceof` требует реальное runtime-значение — например, класс:

```ts
class User {
  constructor(public id: number) {}
}

value instanceof User;
```

Данные из API не становятся безопасными только из-за аннотации:

```ts
const response = await fetch('/api/user');
const user = (await response.json()) as User;
```

`as User` не проверяет объект. Для ненадёжных внешних данных нужна runtime-валидация, например ручной type guard или библиотека схем.

```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'number'
  );
}
```

---

## 16. `any`, `unknown`, `never` и `void`

### `any`

Отключает значительную часть проверки типов:

```ts
let value: any;
value.notExisting.method(); // TypeScript разрешит
```

Использовать следует только осознанно.

### `unknown`

Означает неизвестное значение, которое нужно проверить перед использованием:

```ts
function print(value: unknown) {
  if (typeof value === 'string') {
    console.log(value.toUpperCase());
  }
}
```

Для внешних данных `unknown` безопаснее `any`.

### `never`

Описывает значение, которое невозможно получить:

```ts
function fail(message: string): never {
  throw new Error(message);
}
```

Также применяется для проверки исчерпывающего `switch`:

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}
```

### `void`

Обычно означает, что результат функции не используется:

```ts
type Logger = (message: string) => void;
```

---

## 17. Структурная типизация

TypeScript сравнивает типы в основном по их структуре, а не по имени.

```ts

type User = {
  id: number;
  name: string;
};

type Customer = {
  id: number;
  name: string;
};

const customer: Customer = {
  id: 1,
  name: 'Alex',
};

const user: User = customer; // допустимо
```

Хотя `User` и `Customer` объявлены отдельно, их структуры совместимы.

Если требуется различать одинаковые примитивы по бизнес-смыслу, можно применять branding:

```ts
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };
```

Это паттерн системы типов, а не отдельная runtime-сущность.

---

## 18. Частые ошибки

### Слишком широкие типы

```ts
type Status = string;
```

Если набор значений известен, лучше литеральный union:

```ts
type Status = 'pending' | 'success' | 'error';
```

### Необоснованный `as`

```ts
const user = response as User;
```

Type assertion не преобразует и не валидирует значение, а только сообщает компилятору выбранный тип.

### Чрезмерное использование `any`

`any` распространяется по коду и скрывает ошибки. Для неизвестных данных лучше начинать с `unknown`.

### Сложные пересечения

Пересечение несовместимых типов способно создать невозможные поля типа `never`. Для вариантов состояния чаще нужен union, а не intersection.

### Дублирование типа и значения

Вместо ручного повторения можно вывести тип:

```ts
const roles = ['admin', 'editor', 'viewer'] as const;
type Role = (typeof roles)[number];
```

`Role` будет равен `'admin' | 'editor' | 'viewer'`.

---

## 19. Популярные вопросы на интервью

### Что делает `type`?

Создаёт псевдоним типа. Он может именовать объект, union, intersection, функцию, tuple, примитив или результат преобразования типов.

### Создаёт ли `type` новый тип в runtime?

Нет. Он используется компилятором и удаляется из итогового JavaScript.

### Можно ли проверить `value instanceof SomeType`?

Нет, если `SomeType` объявлен только через `type`. Для runtime-проверки нужен класс, функция-предикат или схема валидации.

### Чем union отличается от intersection?

`A | B` означает, что значение соответствует `A` или `B`. `A & B` требует одновременного соответствия обоим типам.

### Чем `type` отличается от `interface`?

`type` поддерживает union, tuple, conditional и mapped types. `interface` ориентирован на объектные контракты, поддерживает `extends` и declaration merging. Обычный объект можно описать обоими способами.

### Можно ли расширить `type`?

Да, обычно через пересечение:

```ts
type Admin = User & { permissions: string[] };
```

### Можно ли использовать `type` в `implements`?

Да, если итоговый тип описывает статически известную объектную структуру:

```ts
type Named = {
  name: string;
};

class User implements Named {
  constructor(public name: string) {}
}
```

Класс не может реализовать произвольный union, потому что его члены не образуют одну статически известную структуру.

### Что такое type inference?

Это способность TypeScript выводить тип автоматически по значению и контексту:

```ts
const count = 10; // тип выводится автоматически
```

Аннотацию не нужно добавлять там, где вывод компилятора уже точен и понятен.

---

## 20. Практические задачи

### Задача 1: описать успешный и ошибочный ответ

```ts
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };
```

После проверки `success` TypeScript сузит тип до нужного варианта.

### Задача 2: получить тип элемента массива

```ts
const users = [
  { id: 1, name: 'Alex' },
  { id: 2, name: 'Max' },
];

type User = (typeof users)[number];
```

### Задача 3: получить только изменяемые поля DTO

```ts
type User = {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
};

type UpdateUserDto = Partial<Pick<User, 'name' | 'email'>>;
```

### Задача 4: проверить исчерпывающий `switch`

```ts
type Status = 'loading' | 'success' | 'error';

function assertNever(value: never): never {
  throw new Error(`Unknown status: ${value}`);
}

function renderStatus(status: Status) {
  switch (status) {
    case 'loading':
      return 'Loading';
    case 'success':
      return 'Done';
    case 'error':
      return 'Failed';
    default:
      return assertNever(status);
  }
}
```

Если в union добавить новое состояние и не обработать его, TypeScript покажет ошибку в `assertNever(status)`.

---

## 21. Развёрнутый ответ для собеседования

> `type` в TypeScript создаёт псевдоним типа. С его помощью можно дать имя структуре объекта, сигнатуре функции, кортежу, union или intersection, а также строить новые типы через дженерики, `keyof`, mapped и conditional types. Например, `type Status = 'pending' | 'success' | 'error'` ограничивает значение конкретным набором строк.
>
> `type` существует только на этапе статической проверки и после компиляции исчезает, поэтому через него нельзя выполнить runtime-проверку. Данные из API всё равно необходимо валидировать. TypeScript использует структурную типизацию: совместимость определяется главным образом набором свойств, а не именем псевдонима.
>
> Обычный объект можно описать и через `type`, и через `interface`. `interface` удобен для расширяемых объектных контрактов и поддерживает declaration merging, а `type` универсальнее: он позволяет напрямую описывать union, tuple, conditional и mapped types. В реальном проекте выбор также зависит от соглашений команды.

---

## 22. Чек-лист перед интервью

- Объяснить, что `type` создаёт псевдоним типа.
- Показать объект, union, intersection, функцию и tuple.
- Объяснить narrowing для union.
- Уметь построить discriminated union.
- Рассказать отличия `type` и `interface`.
- Объяснить, почему типы отсутствуют в runtime.
- Отличать `any`, `unknown`, `never` и `void`.
- Использовать `keyof`, type-level `typeof` и indexed access.
- Знать основные utility types: `Partial`, `Pick`, `Omit`, `Record`.
- Понимать дженерики, mapped и conditional types.
- Помнить, что `as` не валидирует и не преобразует данные.
- Объяснить структурную типизацию TypeScript.
