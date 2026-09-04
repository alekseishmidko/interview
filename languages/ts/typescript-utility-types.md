# Utility types в TypeScript — шпаргалка для собеседования

## Краткий ответ

**Utility types** — встроенные обобщённые типы TypeScript, которые создают новый тип на основе существующего: делают свойства необязательными или обязательными, выбирают и исключают поля, преобразуют union либо извлекают параметры и результаты функций.

```ts
interface User {
  id: string;
  name: string;
  email: string;
  active: boolean;
}

type CreateUserDto = Omit<User, 'id'>;

type UpdateUserDto = Partial<
  Pick<User, 'name' | 'email' | 'active'>
>;

type PublicUser = Pick<User, 'id' | 'name'>;
```

Utility types:

- работают только на уровне типов;
- не изменяют runtime-объекты;
- не валидируют данные;
- обычно построены через generics, `keyof`, mapped types, conditional types и `infer`.

---

## 1. Зачем нужны utility types

Без утилит связанные модели приходится дублировать:

```ts
interface User {
  id: string;
  name: string;
  email: string;
}

interface UpdateUserDto {
  name?: string;
  email?: string;
}
```

Если `User` изменится, производная модель может устареть. Utility types позволяют явно вывести её из источника:

```ts
type UpdateUserDto = Partial<
  Pick<User, 'name' | 'email'>
>;
```

Преимущества:

- меньше дублирования;
- модели изменяются согласованно;
- намерение выражается через тип;
- проще создавать DTO и публичные представления;
- можно комбинировать преобразования.

Недостаток: слишком длинная цепочка utility types иногда читается хуже отдельного именованного интерфейса.

---

## 2. Основные группы

| Группа | Utility types |
| --- | --- |
| Изменение свойств | `Partial`, `Required`, `Readonly` |
| Выбор свойств | `Pick`, `Omit` |
| Словари | `Record` |
| Работа с union | `Exclude`, `Extract`, `NonNullable` |
| Функции | `Parameters`, `ReturnType` |
| Конструкторы | `ConstructorParameters`, `InstanceType` |
| Promise | `Awaited` |
| `this` | `ThisParameterType`, `OmitThisParameter`, `ThisType` |
| Строковые литералы | `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` |
| Управление inference | `NoInfer` |

---

## 3. `Partial<T>`

Делает все свойства объекта необязательными.

```ts
interface User {
  id: string;
  name: string;
  email: string;
}

type UserPatch = Partial<User>;
```

Результат концептуально выглядит так:

```ts
type UserPatch = {
  id?: string;
  name?: string;
  email?: string;
};
```

Пример применения:

```ts
function updateUser(
  user: User,
  patch: Partial<User>,
): User {
  return { ...user, ...patch };
}
```

Упрощённая реализация:

```ts
type MyPartial<T> = {
  [Key in keyof T]?: T[Key];
};
```

### Важное ограничение

`Partial` работает поверхностно:

```ts
interface Settings {
  profile: {
    theme: string;
    language: string;
  };
}

type SettingsPatch = Partial<Settings>;
```

`profile` можно не передавать, но если передать, его внутренние обязательные свойства не становятся автоматически необязательными:

```ts
const patch: SettingsPatch = {
  // profile: { theme: 'dark' }
  // Ошибка: отсутствует language
};
```

---

## 4. `Required<T>`

Делает все необязательные свойства обязательными.

```ts
interface Config {
  apiUrl?: string;
  timeout?: number;
}

type CompleteConfig = Required<Config>;
```

```ts
const config: CompleteConfig = {
  apiUrl: '/api',
  timeout: 5000,
};
```

Упрощённая реализация:

```ts
type MyRequired<T> = {
  [Key in keyof T]-?: T[Key];
};
```

Модификатор `-?` удаляет необязательность свойства.

`Required` не заполняет отсутствующие значения в runtime — он только меняет требования компилятора.

---

## 5. `Readonly<T>`

Делает все свойства доступными только для чтения через данный тип.

```ts
interface User {
  id: string;
  name: string;
}

type ReadonlyUser = Readonly<User>;

const user: ReadonlyUser = {
  id: '1',
  name: 'Alex',
};

// user.name = 'Max'; // ошибка TypeScript
```

Упрощённая реализация:

```ts
type MyReadonly<T> = {
  readonly [Key in keyof T]: T[Key];
};
```

### Важные ограничения

`Readonly`:

- работает поверхностно;
- не вызывает `Object.freeze()`;
- не гарантирует неизменность объекта через другую mutable-ссылку;
- удаляется из JavaScript.

```ts
interface State {
  nested: {
    count: number;
  };
}

const state: Readonly<State> = {
  nested: { count: 0 },
};

state.nested.count += 1; // допустимо: nested-свойства не стали readonly
```

---

## 6. `Pick<T, K>`

Создаёт тип только из указанных свойств.

```ts
interface User {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
}

type UserPreview = Pick<User, 'id' | 'name'>;
```

Результат:

```ts
type UserPreview = {
  id: string;
  name: string;
};
```

Упрощённая реализация:

```ts
type MyPick<
  T,
  K extends keyof T,
> = {
  [Key in K]: T[Key];
};
```

`K extends keyof T` запрещает выбирать несуществующие ключи:

```ts
// type Wrong = Pick<User, 'age'>; // ошибка
```

`Pick` полезен для:

- публичного представления объекта;
- props компонента;
- DTO с ограниченным набором полей;
- параметров функций;
- переиспользования части модели.

---

## 7. `Omit<T, K>`

Создаёт тип, исключая указанные свойства.

```ts
type PublicUser = Omit<
  User,
  'passwordHash' | 'email'
>;
```

Упрощённая реализация:

```ts
type MyOmit<
  T,
  K extends PropertyKey,
> = Pick<T, Exclude<keyof T, K>>;
```

Логика:

1. `keyof T` получает все ключи.
2. `Exclude` удаляет `K`.
3. `Pick` собирает объект из оставшихся ключей.

Пример DTO:

```ts
type CreateUserDto = Omit<
  User,
  'id' | 'passwordHash'
>;
```

### Важное замечание

`Omit` не удаляет свойства из runtime-объекта:

```ts
const publicUser: PublicUser = fullUser;
```

Если `fullUser` содержит `passwordHash`, это свойство физически останется в объекте. Тип лишь запрещает обращаться к нему через ссылку `PublicUser`.

Для реального удаления нужно создать новый объект:

```ts
const { passwordHash, ...publicUser } = fullUser;
```

---

## 8. `Pick` vs `Omit`

```ts
type PublicByPick = Pick<
  User,
  'id' | 'name'
>;

type PublicByOmit = Omit<
  User,
  'email' | 'passwordHash'
>;
```

| `Pick` | `Omit` |
| --- | --- |
| Перечисляет нужные поля | Перечисляет исключаемые поля |
| Безопасен для минимального публичного контракта | Удобен, когда исключений мало |
| Новое поле источника не попадёт автоматически | Новое поле источника попадёт автоматически |

Для безопасности публичного API часто предпочтительнее `Pick`: добавленное позже секретное поле не появится в публичном типе автоматически.

---

## 9. `Record<K, T>`

Создаёт объектный тип с ключами `K` и значениями `T`.

```ts
type Role = 'admin' | 'editor' | 'viewer';

type Permissions = Record<Role, string[]>;
```

```ts
const permissions: Permissions = {
  admin: ['read', 'write', 'delete'],
  editor: ['read', 'write'],
  viewer: ['read'],
};
```

Если забыть ключ, TypeScript сообщит об ошибке. Это делает `Record` полезным для exhaustive mapping.

Упрощённая реализация:

```ts
type MyRecord<
  K extends PropertyKey,
  Value,
> = {
  [Key in K]: Value;
};
```

`PropertyKey` равен:

```ts
string | number | symbol
```

### `Record<string, T>`

```ts
type UsersById = Record<string, User>;
```

Этот тип утверждает, что по любому строковому ключу будет `User`. В runtime ключ может отсутствовать. При включённом `noUncheckedIndexedAccess` индексированный доступ учитывает `undefined` точнее.

```ts
const user = usersById['unknown-id'];
// С noUncheckedIndexedAccess: User | undefined
```

---

## 10. `Exclude<T, U>`

Удаляет из union `T` варианты, совместимые с `U`.

```ts
type Status =
  | 'idle'
  | 'loading'
  | 'success'
  | 'error';

type FinishedStatus = Exclude<
  Status,
  'idle' | 'loading'
>;
// 'success' | 'error'
```

Упрощённая реализация:

```ts
type MyExclude<T, U> =
  T extends U ? never : T;
```

Это distributive conditional type:

1. каждый член `T` проверяется отдельно;
2. совпавший заменяется на `never`;
3. `never` исчезает из union.

`Exclude` работает с членами union, а не с полями объекта. Для полей применяется `Omit`.

---

## 11. `Extract<T, U>`

Оставляет только варианты `T`, совместимые с `U`.

```ts
type Value =
  | string
  | number
  | (() => void);

type FunctionValue = Extract<Value, Function>;
// () => void
```

Упрощённая реализация:

```ts
type MyExtract<T, U> =
  T extends U ? T : never;
```

Пример с discriminated union:

```ts
type AppEvent =
  | { type: 'user.created'; user: User }
  | { type: 'user.deleted'; userId: string }
  | { type: 'order.created'; orderId: string };

type UserCreatedEvent = Extract<
  AppEvent,
  { type: 'user.created' }
>;
```

---

## 12. `Exclude` vs `Extract`

```ts
type Union = 'a' | 'b' | 'c';

type WithoutA = Exclude<Union, 'a'>;
// 'b' | 'c'

type OnlyA = Extract<Union, 'a'>;
// 'a'
```

| Utility | Поведение |
| --- | --- |
| `Exclude<T, U>` | Удаляет совместимые с `U` варианты |
| `Extract<T, U>` | Оставляет совместимые с `U` варианты |

---

## 13. `NonNullable<T>`

Удаляет `null` и `undefined` из типа.

```ts
type Value =
  string | number | null | undefined;

type DefinedValue = NonNullable<Value>;
// string | number
```

Концептуальная реализация:

```ts
type MyNonNullable<T> =
  T extends null | undefined
    ? never
    : T;
```

Использование с полем:

```ts
interface ApiResponse {
  data?: User[] | null;
}

type Users = NonNullable<ApiResponse['data']>;
// User[]
```

`NonNullable` не проверяет значение в runtime. Переменную всё равно нужно сузить условием.

---

## 14. `Parameters<T>`

Возвращает параметры функции в виде tuple.

```ts
function createOrder(
  userId: string,
  amount: number,
  express: boolean,
) {
  // ...
}

type CreateOrderParams = Parameters<
  typeof createOrder
>;
// [userId: string, amount: number, express: boolean]
```

Упрощённая реализация:

```ts
type MyParameters<
  T extends (...args: any[]) => any,
> = T extends (...args: infer Params) => any
  ? Params
  : never;
```

Пример wrapper-функции:

```ts
function withLogging<
  Fn extends (...args: any[]) => any,
>(fn: Fn) {
  return (...args: Parameters<Fn>): ReturnType<Fn> => {
    console.log(args);
    return fn(...args);
  };
}
```

---

## 15. `ReturnType<T>`

Извлекает возвращаемый тип функции.

```ts
function createUser() {
  return {
    id: '1',
    name: 'Alex',
  };
}

type User = ReturnType<typeof createUser>;
```

Упрощённая реализация:

```ts
type MyReturnType<
  T extends (...args: any[]) => any,
> = T extends (...args: any[]) => infer Result
  ? Result
  : never;
```

Для async-функции вернётся `Promise<...>`:

```ts
async function loadUser() {
  return { id: '1', name: 'Alex' };
}

type Result = ReturnType<typeof loadUser>;
// Promise<{ id: string; name: string }>
```

Чтобы получить внутренний результат:

```ts
type User = Awaited<ReturnType<typeof loadUser>>;
```

При перегруженной функции `ReturnType` обычно использует последнюю, наиболее общую сигнатуру.

---

## 16. `ConstructorParameters<T>`

Получает параметры конструктора в виде tuple.

```ts
class User {
  constructor(
    public id: string,
    public name: string,
  ) {}
}

type UserConstructorArgs =
  ConstructorParameters<typeof User>;
// [id: string, name: string]
```

Упрощённая реализация:

```ts
type MyConstructorParameters<
  T extends abstract new (...args: any[]) => any,
> = T extends abstract new (...args: infer Params) => any
  ? Params
  : never;
```

---

## 17. `InstanceType<T>`

Получает тип экземпляра конструктора.

```ts
class UserService {
  getUser() {
    return { id: '1' };
  }
}

type Service = InstanceType<typeof UserService>;
// UserService
```

Упрощённая реализация:

```ts
type MyInstanceType<
  T extends abstract new (...args: any[]) => any,
> = T extends abstract new (...args: any[]) => infer Instance
  ? Instance
  : never;
```

Не путать:

```ts
UserService        // тип экземпляра в type-позиции
typeof UserService // тип конструктора класса
```

В `InstanceType` передают именно тип конструктора.

---

## 18. `Awaited<T>`

Рекурсивно получает значение Promise или Promise-подобного объекта.

```ts
type A = Awaited<Promise<string>>;
// string

type B = Awaited<Promise<Promise<number>>>;
// number

type C = Awaited<string>;
// string
```

Практический пример:

```ts
async function fetchUsers() {
  return [{ id: '1', name: 'Alex' }];
}

type Users = Awaited<
  ReturnType<typeof fetchUsers>
>;
```

`Awaited` моделирует поведение `await` точнее, чем самодельный одноуровневый `UnwrapPromise`.

---

## 19. `ThisParameterType<T>`

Извлекает тип явно объявленного параметра `this` функции.

```ts
function greet(
  this: { name: string },
  message: string,
) {
  return `${message}, ${this.name}`;
}

type Context = ThisParameterType<typeof greet>;
// { name: string }
```

Параметр `this` существует только для типизации и не попадает в runtime-список аргументов.

Если явного `this` нет, результатом обычно будет `unknown`.

---

## 20. `OmitThisParameter<T>`

Удаляет явный параметр `this` из типа функции.

```ts
function greet(
  this: { name: string },
  message: string,
) {
  return `${message}, ${this.name}`;
}

const bound = greet.bind({ name: 'Alex' });

type BoundGreet = OmitThisParameter<typeof greet>;
// (message: string) => string
```

Полезен при типизации привязанных функций и методов.

---

## 21. `ThisType<T>`

`ThisType<T>` — marker utility для контекстной типизации `this` внутри объектного литерала. Он не возвращает преобразованную структуру как `Partial` или `Pick`.

```ts
type ObjectDescriptor<Data, Methods> = {
  data?: Data;
  methods?: Methods & ThisType<Data & Methods>;
};

function makeObject<Data, Methods>(
  descriptor: ObjectDescriptor<Data, Methods>,
): Data & Methods {
  return {
    ...descriptor.data,
    ...descriptor.methods,
  } as Data & Methods;
}
```

```ts
const object = makeObject({
  data: {
    x: 0,
    y: 0,
  },
  methods: {
    move(dx: number, dy: number) {
      this.x += dx;
      this.y += dy;
    },
  },
});
```

Для корректной проверки `this` важна настройка `noImplicitThis`, обычно включаемая через `strict`.

`ThisType<T>` используется реже остальных и чаще встречается в библиотечных API.

---

## 22. Строковые utility types

Эти intrinsic-типы преобразуют строковые литеральные типы.

### `Uppercase<S>`

```ts
type Method = Uppercase<'get' | 'post'>;
// 'GET' | 'POST'
```

### `Lowercase<S>`

```ts
type Method = Lowercase<'GET' | 'POST'>;
// 'get' | 'post'
```

### `Capitalize<S>`

```ts
type Event = Capitalize<'click'>;
// 'Click'
```

### `Uncapitalize<S>`

```ts
type Event = Uncapitalize<'Submit'>;
// 'submit'
```

Комбинация с template literal types:

```ts
type EventName = 'click' | 'submit';

type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onSubmit'
```

Это compile-time-преобразование типов. Строковое runtime-значение автоматически не изменяется.

---

## 23. `NoInfer<T>`

`NoInfer<T>` блокирует вывод generic-типа из конкретной позиции, сохраняя проверку совместимости.

```ts
function createStreetLight<C extends string>(
  colors: C[],
  defaultColor?: NoInfer<C>,
) {
  // ...
}
```

```ts
createStreetLight(
  ['red', 'yellow', 'green'],
  'red',
); // допустимо

createStreetLight(
  ['red', 'yellow', 'green'],
  // 'blue', // ошибка
);
```

Без `NoInfer` TypeScript мог бы использовать `defaultColor` как дополнительный источник вывода `C` и расширить union значением `'blue'`.

Не путать:

- `infer` извлекает тип внутри conditional type;
- `NoInfer<T>` запрещает конкретной позиции влиять на generic inference.

---

## 24. Композиция utility types

Utility types часто используются вместе.

```ts
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}
```

### DTO создания

```ts
type CreateUserDto = Omit<
  User,
  'id' | 'createdAt'
>;
```

### DTO обновления

```ts
type UpdateUserDto = Partial<
  Pick<User, 'name' | 'email' | 'role'>
>;
```

### Публичная модель

```ts
type PublicUser = Readonly<
  Pick<User, 'id' | 'name' | 'role'>
>;
```

### Обязательная конфигурация

```ts
type ResolvedConfig = Readonly<Required<Config>>;
```

Если цепочка становится длинной, лучше дать промежуточным типам имена:

```ts
type EditableUserFields = Pick<
  User,
  'name' | 'email' | 'role'
>;

type UpdateUserDto = Partial<EditableUserFields>;
```

---

## 25. Utility types и `interface`

Utility type может принимать интерфейс:

```ts
interface User {
  id: string;
  name: string;
}

type UserPatch = Partial<User>;
```

Результат utility type обычно сохраняют через `type`, поскольку выражения вроде `Partial<User>` уже являются типовыми вычислениями.

Если нужен интерфейс, расширяющий объектный результат с известными полями, это иногда возможно:

```ts
interface UserWithOptionalName
  extends Partial<Pick<User, 'name'>> {
  id: string;
}
```

Но для производных моделей часто проще и понятнее использовать type alias:

```ts
type UserWithOptionalName =
  Partial<Pick<User, 'name'>> & {
    id: string;
  };
```

---

## 26. Собственные utility types

Встроенного набора не всегда достаточно.

### `ValueOf<T>`

Получает union значений объекта:

```ts
type ValueOf<T> = T[keyof T];

type Statuses = {
  pending: 'PENDING';
  success: 'SUCCESS';
  error: 'ERROR';
};

type Status = ValueOf<Statuses>;
// 'PENDING' | 'SUCCESS' | 'ERROR'
```

### `Nullable<T>`

```ts
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};
```

### `Mutable<T>`

```ts
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};
```

### `RequireKeys<T, K>`

Делает выбранные ключи обязательными:

```ts
type RequireKeys<
  T,
  K extends keyof T,
> = Omit<T, K> & Required<Pick<T, K>>;
```

### `OptionalKeys<T, K>`

Делает выбранные ключи необязательными:

```ts
type OptionalKeys<
  T,
  K extends keyof T,
> = Omit<T, K> & Partial<Pick<T, K>>;
```

---

## 27. `DeepPartial<T>` и его сложности

Наивная реализация:

```ts
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? DeepPartial<T[K]>
    : T[K];
};
```

Она кажется удобной, но тип `object` включает:

- массивы;
- функции;
- `Date`;
- `Map`;
- `Set`;
- экземпляры классов.

Для них рекурсивное преобразование может дать нежелательный результат. Более практичная версия должна отдельно обрабатывать примитивы, функции, массивы и специальные контейнеры.

```ts
type Primitive =
  | string
  | number
  | boolean
  | bigint
  | symbol
  | null
  | undefined;

type DeepPartial<T> =
  T extends Primitive | Function
    ? T
    : T extends readonly (infer Item)[]
      ? readonly DeepPartial<Item>[]
      : T extends object
        ? { [K in keyof T]?: DeepPartial<T[K]> }
        : T;
```

Даже эта версия определяет конкретную семантику массивов и не обязана подходить всем проектам. Глубокие utilities лучше проектировать под реальную модель данных, а не копировать без проверки.

---

## 28. Utility types не меняют runtime

```ts
type PublicUser = Omit<User, 'passwordHash'>;

const publicUser: PublicUser = fullUser;
```

Поле `passwordHash` остаётся в объекте. Тип только скрывает его от текущей типизированной ссылки.

Аналогично:

- `Readonly<T>` не вызывает `Object.freeze()`;
- `Required<T>` не добавляет отсутствующие поля;
- `Partial<T>` не удаляет обязательные свойства объекта;
- `NonNullable<T>` не проверяет значение;
- `Uppercase<S>` не изменяет строку.

Для runtime-поведения нужен JavaScript-код.

---

## 29. Utility types не валидируют API

```ts
type CreateUserDto = Omit<User, 'id'>;

const data = (await response.json()) as CreateUserDto;
```

TypeScript не проверяет фактическую структуру JSON. Utility types только формируют ожидаемый статический тип.

Для внешних данных нужны:

- ручной type guard;
- assertion function с реальной проверкой;
- библиотека runtime-схем;
- серверная валидация.

```ts
function isCreateUserDto(
  value: unknown,
): value is CreateUserDto {
  return (
    typeof value === 'object' &&
    value !== null &&
    'name' in value &&
    typeof value.name === 'string'
  );
}
```

---

## 30. Частые ошибки

### Ожидать глубокое преобразование

`Partial`, `Required` и `Readonly` поверхностные.

### Использовать `Omit` как runtime-удаление

`Omit` не удаляет поля из объекта и не защищает секретные данные при сериализации.

### Путать `Exclude` и `Omit`

- `Exclude` фильтрует union;
- `Omit` исключает ключи объекта.

```ts
type WithoutError = Exclude<Status, 'error'>;
type WithoutPassword = Omit<User, 'passwordHash'>;
```

### Использовать `Record<string, T>` как гарантию существования ключа

Runtime-ключ может отсутствовать. Включайте `noUncheckedIndexedAccess` или проверяйте значение.

### Создавать DTO из database entity без анализа

Механический `Partial<Entity>` способен разрешить изменение служебных полей вроде `id`, `createdAt`, `role` или `passwordHash`. Сначала выбирайте допустимые поля через `Pick`, затем применяйте `Partial`.

```ts
type UpdateUserDto = Partial<
  Pick<User, 'name' | 'email'>
>;
```

### Слишком сложная композиция

```ts
type Result = Readonly<
  Partial<
    Omit<
      Pick<User, SomeKeys>,
      OtherKeys
    >
  >
>;
```

Такой код лучше разбить на именованные этапы.

### Дублировать источник истины в обе стороны

Не каждую модель нужно выводить из другой. Если API DTO и database entity меняются независимо, связывать их через сложные utilities может быть вредно.

---

## 31. Популярные вопросы на интервью

### Что такое utility types?

Встроенные generic-типы TypeScript для преобразования и извлечения типов. Они уменьшают дублирование и работают только во время компиляции.

### Какие utility types вы используете чаще всего?

`Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `NonNullable`, `Parameters`, `ReturnType` и `Awaited`.

### Чем `Pick` отличается от `Omit`?

`Pick` оставляет перечисленные поля, `Omit` оставляет все, кроме перечисленных.

### Чем `Exclude` отличается от `Omit`?

`Exclude` удаляет варианты из union, а `Omit` удаляет ключи из объектного типа.

### Является ли `Partial<T>` глубоким?

Нет. Он делает необязательными только свойства верхнего уровня.

### Является ли `Readonly<T>` runtime-защитой?

Нет. Это поверхностное ограничение системы типов. Для runtime-заморозки используется `Object.freeze`, причём он тоже имеет собственные ограничения глубины.

### Что возвращает `ReturnType` для async-функции?

`Promise<Result>`. Для получения внутреннего результата применяется `Awaited<ReturnType<typeof fn>>`.

### Что возвращает `Parameters<T>`?

Tuple параметров функции, включая имена параметров, если TypeScript их сохранил для отображения.

### Для чего нужен `Record`?

Для описания объекта с заданным набором ключей и единым типом значений. С конечным union ключей он обеспечивает проверку полноты mapping.

### Что делает `NonNullable<T>`?

Удаляет `null` и `undefined` из типа.

### Как работают `Exclude` и `Extract`?

Это distributive conditional types. Они проверяют отдельно каждый член union и заменяют неподходящие варианты на `never`.

### Что делает `NoInfer<T>`?

Запрещает конкретной позиции участвовать в выводе generic-типа, но сохраняет проверку значения на соответствие уже выведенному типу.

### Можно ли написать собственный utility type?

Да. Для этого используют generics, `keyof`, indexed access, mapped types, conditional types и `infer`.

### Существуют ли utility types в runtime?

Нет. Они полностью удаляются при компиляции.

---

## 32. Практические задачи с решениями

### Задача 1: DTO создания пользователя

```ts
interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}
```

**Решение:**

```ts
type CreateUserDto = Omit<
  User,
  'id' | 'createdAt'
>;
```

### Задача 2: DTO частичного обновления

Разрешено менять только `name` и `email`.

```ts
type UpdateUserDto = Partial<
  Pick<User, 'name' | 'email'>
>;
```

### Задача 3: обязательные настройки

```ts
interface Config {
  apiUrl?: string;
  timeout?: number;
}

type ResolvedConfig = Required<Config>;
```

### Задача 4: mapping всех статусов

```ts
type Status = 'pending' | 'success' | 'error';

const labels: Record<Status, string> = {
  pending: 'Ожидание',
  success: 'Готово',
  error: 'Ошибка',
};
```

### Задача 5: убрать nullable

```ts
type Value = string | null | undefined;

type DefinedValue = NonNullable<Value>;
// string
```

### Задача 6: получить данные async-функции

```ts
async function getOrders() {
  return [{ id: '1', amount: 100 }];
}

type Orders = Awaited<
  ReturnType<typeof getOrders>
>;
// { id: string; amount: number }[]
```

### Задача 7: обёртка с теми же аргументами

```ts
function withLog<
  Fn extends (...args: any[]) => any,
>(fn: Fn) {
  return (...args: Parameters<Fn>): ReturnType<Fn> => {
    console.log('Arguments:', args);
    return fn(...args);
  };
}
```

### Задача 8: сделать выбранное поле обязательным

```ts
type RequireKeys<
  T,
  K extends keyof T,
> = Omit<T, K> & Required<Pick<T, K>>;

interface Draft {
  title?: string;
  description?: string;
}

type DraftWithTitle = RequireKeys<Draft, 'title'>;
```

### Задача 9: оставить события пользователя

```ts
type AppEvent =
  | { type: 'user.created'; id: string }
  | { type: 'user.deleted'; id: string }
  | { type: 'order.created'; id: string };

type UserEvent = Extract<
  AppEvent,
  { type: `user.${string}` }
>;
```

### Задача 10: получить имена обработчиков

```ts
type EventName = 'click' | 'submit' | 'change';

type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onSubmit' | 'onChange'
```

---

## 33. Готовый развёрнутый ответ для собеседования

> Utility types — это встроенные generic-типы TypeScript, которые преобразуют или извлекают другие типы. Например, `Partial<T>` делает свойства необязательными, `Required<T>` — обязательными, `Readonly<T>` запрещает переназначение, `Pick<T, K>` выбирает поля, а `Omit<T, K>` исключает их. Для словарей используется `Record<K, V>`.
>
> Для union есть `Exclude`, `Extract` и `NonNullable`. Они построены на distributive conditional types и `never`. Для функций применяются `Parameters` и `ReturnType`, для конструкторов — `ConstructorParameters` и `InstanceType`, а внутренний результат Promise можно получить через `Awaited`.
>
> Utility types работают только при компиляции: `Omit` не удаляет поле из runtime-объекта, `Readonly` не вызывает `Object.freeze`, а `Partial` не изменяет данные. Большинство объектных utilities поверхностные. В реальном проекте я комбинирую их для DTO, но сначала явно ограничиваю разрешённые поля через `Pick`, чтобы случайно не открыть изменение служебных данных.

---

## 34. Мини-шпаргалка

```ts
Partial<T>               // все поля необязательные
Required<T>              // все поля обязательные
Readonly<T>              // все поля readonly
Pick<T, K>               // оставить поля K
Omit<T, K>               // исключить поля K
Record<K, V>             // объект: ключи K, значения V
Exclude<T, U>            // убрать U из union T
Extract<T, U>            // оставить U в union T
NonNullable<T>           // убрать null | undefined
Parameters<F>            // tuple параметров функции
ReturnType<F>            // возвращаемый тип функции
ConstructorParameters<C> // tuple параметров конструктора
InstanceType<C>          // экземпляр конструктора
Awaited<T>               // результат await
ThisParameterType<F>     // тип this функции
OmitThisParameter<F>     // функция без параметра this
Uppercase<S>             // строка в верхнем регистре
Lowercase<S>             // строка в нижнем регистре
Capitalize<S>            // первая буква заглавная
Uncapitalize<S>          // первая буква строчная
NoInfer<T>               // блокировать источник inference
```

---

## 35. Чек-лист перед интервью

- Объяснить назначение utility types.
- Самостоятельно реализовать `Partial`, `Required`, `Readonly` и `Pick`.
- Объяснить реализацию `Omit` через `Pick` и `Exclude`.
- Отличать `Pick` от `Omit`.
- Отличать `Exclude` от `Omit`.
- Рассказать о поверхностном характере объектных utilities.
- Объяснить runtime-ограничения.
- Использовать `Record` для конечного union ключей.
- Реализовать `Exclude`, `Extract` и `NonNullable`.
- Использовать `Parameters` и `ReturnType`.
- Получить результат async-функции через `Awaited<ReturnType<...>>`.
- Различать тип экземпляра и тип конструктора.
- Знать utilities для `this` и строковых литералов.
- Объяснить отличие `NoInfer` от `infer`.
- Уметь составить безопасный `UpdateDto` через `Partial<Pick<...>>`.
