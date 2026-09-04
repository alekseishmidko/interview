# Структурная типизация и Excess Property Checking в TypeScript

## Короткий ответ для собеседования

TypeScript использует преимущественно **структурную типизацию**: совместимость типов определяется их структурой, а не именем или явным наследованием. Если значение содержит все обязательные свойства целевого типа с совместимыми типами, его можно присвоить этому типу.

```ts
type User = {
  id: number;
  name: string;
};

const employee = {
  id: 1,
  name: 'Alex',
  department: 'Frontend',
};

const user: User = employee; // корректно
```

`employee` совместим с `User`, потому что имеет как минимум `id: number` и `name: string`. Дополнительное поле `department` обычно не мешает.

Но для **свежих объектных литералов** TypeScript выполняет **excess property checking** — дополнительную проверку неизвестных свойств:

```ts
const user: User = {
  id: 1,
  name: 'Alex',
  department: 'Frontend', // ошибка: неизвестное свойство
};
```

Excess property checking помогает находить опечатки и ошибочные поля, но не делает типы «точными» и не является runtime-валидацией.

---

## 1. Что такое структурная типизация

При структурной типизации TypeScript отвечает на вопрос:

> Обладает ли исходное значение всеми возможностями, которые требует целевой тип?

```ts
interface Named {
  name: string;
}

const developer = {
  name: 'Anna',
  language: 'TypeScript',
};

const named: Named = developer; // корректно
```

Тип переменной `developer` не называется `Named` и не объявляет `implements Named`. Совпадения структуры достаточно.

Эту идею часто связывают с принципом **duck typing**:

> Если объект выглядит как нужный тип и ведёт себя как нужный тип, его можно использовать как этот тип.

Разница в том, что TypeScript в основном проверяет такую совместимость статически, до запуска программы.

---

## 2. Structural typing против nominal typing

В номинальной системе совместимость определяется именем типа или явной связью наследования.

Условный номинальный подход:

```ts
class UserId {}
class OrderId {}

// Типы различаются из-за своей декларации,
// даже если их устройство одинаково.
```

В TypeScript два одинаково устроенных объектных типа обычно совместимы:

```ts
type Point = {
  x: number;
  y: number;
};

type Coordinates = {
  x: number;
  y: number;
};

const point: Point = { x: 10, y: 20 };
const coordinates: Coordinates = point; // корректно
```

| Структурная типизация | Номинальная типизация |
|---|---|
| Сравнивает форму типов | Сравнивает идентичность типов |
| Явное наследование необязательно | Обычно нужна декларативная связь |
| Удобна для JavaScript-объектов | Сильнее разделяет доменные сущности |
| Гибкая композиция | Сложнее случайно смешать одинаковые структуры |

---

## 3. Правило совместимости объектов

При присваивании `source` в `target` проверяются требования целевого типа:

```ts
type Target = {
  id: number;
};

const source = {
  id: 1,
  name: 'Alex',
};

const target: Target = source;
```

У `source` есть обязательное поле `id` подходящего типа. Поле `name` не мешает.

Удобная формулировка:

> У исходного типа должно быть не меньше возможностей, чем требуется целевому.

### Недостающее поле

```ts
type User = {
  id: number;
  name: string;
};

const value = {
  id: 1,
};

const user: User = value;
// Ошибка: отсутствует name
```

### Несовместимый тип поля

```ts
const value = {
  id: '1',
  name: 'Alex',
};

const user: User = value;
// Ошибка: string несовместим с number
```

### Вложенные объекты

Совместимость проверяется рекурсивно:

```ts
type User = {
  profile: {
    email: string;
  };
};

const value = {
  profile: {
    email: 'alex@example.com',
    verified: true,
  },
};

const user: User = value; // корректно
```

---

## 4. Структурная типизация функций

Функции тоже сравниваются по структуре сигнатур.

### Можно игнорировать аргументы callback

```ts
const numbers = [1, 2, 3];

numbers.forEach(value => {
  console.log(value);
});
```

`forEach` передаёт несколько аргументов, но callback имеет право использовать только первый.

```ts
type Handler = (
  value: string,
  index: number,
) => void;

const handler: Handler = value => {
  console.log(value);
};
```

Функция с меньшим числом обязательных параметров может быть совместима с обработчиком, которому при вызове передадут больше аргументов: лишние аргументы JavaScript можно проигнорировать.

Обратное направление небезопасно:

```ts
type OneArgument = (value: string) => void;

const needsTwo = (value: string, index: number) => {};

// const fn: OneArgument = needsTwo;
// Ошибка: вызывающий код не обязан передать index
```

### Возвращаемые значения

Возвращаемый объект может содержать дополнительные поля:

```ts
type CreateUser = () => {
  id: number;
};

const createUser: CreateUser = () => ({
  id: 1,
  name: 'Alex',
});
```

Вызывающей стороне гарантирован `id`, а дополнительный `name` не нарушает контракт.

### Вариантность параметров

Совместимость параметров функций зависит от позиции и настроек компилятора. При `strictFunctionTypes` TypeScript строже проверяет функции как свойства, но методы исторически ведут себя менее строго. Для интервью достаточно помнить:

- результат функции должен быть совместим с ожидаемым результатом;
- callback может игнорировать переданные аргументы;
- функция не может требовать аргумент, который вызывающая сторона не обещает передать;
- при работе с наследованием параметров нужно учитывать контравариантность и `strictFunctionTypes`.

---

## 5. Интерфейсы и классы тоже структурны

Класс может реализовать интерфейс без явного `implements`, если структура совпадает:

```ts
interface Printable {
  print(): void;
}

class Report {
  print() {
    console.log('Report');
  }
}

function run(value: Printable) {
  value.print();
}

run(new Report()); // корректно
```

`implements` выполняет проверку класса, но не меняет его runtime-поведение и не создаёт номинальную идентичность:

```ts
class Invoice implements Printable {
  print() {
    console.log('Invoice');
  }
}
```

И `Report`, и `Invoice` совместимы с `Printable` благодаря методу `print`.

---

## 6. Исключение: `private` и `protected`

Классы с приватными или защищёнными членами получают частично номинальное поведение. Для совместимости такие члены должны происходить из одной декларации.

```ts
class User {
  private id = 1;
}

class Product {
  private id = 1;
}

const user = new User();
const product = new Product();

// const value: User = product;
// Ошибка: private id имеет другое происхождение
```

Хотя публичная форма экземпляров похожа, разные приватные декларации делают классы несовместимыми.

Наследник сохраняет происхождение приватного члена:

```ts
class Admin extends User {}

const value: User = new Admin(); // корректно
```

JavaScript-поля `#private` также влияют на совместимость классов.

---

## 7. Что такое Excess Property Checking

**Excess Property Checking (EPC)** — дополнительная проверка лишних свойств у свежего объектного литерала, когда TypeScript знает ожидаемый объектный тип.

```ts
type User = {
  id: number;
  name: string;
};

const user: User = {
  id: 1,
  name: 'Alex',
  isAdmin: true,
  // Ошибка: isAdmin не объявлено в User
};
```

Зачем это нужно:

- находить опечатки;
- обнаруживать неправильное имя поля;
- предупреждать о неверном понимании API;
- не позволять передать конфигурацию, которую функция проигнорирует.

```ts
type RequestOptions = {
  timeout: number;
};

function request(options: RequestOptions) {}

request({
  timeout: 1000,
  timeuot: 2000,
  // Ошибка помогает найти опечатку
});
```

---

## 8. Почему прямой литерал ошибочен, а переменная разрешена

Прямая передача свежего литерала:

```ts
function printUser(user: User) {}

printUser({
  id: 1,
  name: 'Alex',
  role: 'admin',
  // Ошибка excess property checking
});
```

Через переменную:

```ts
const admin = {
  id: 1,
  name: 'Alex',
  role: 'admin',
};

printUser(admin); // корректно
```

Во втором случае TypeScript применяет обычную структурную совместимость. `admin` уже имеет выведенный тип, а `User` требует лишь наличие совместимых `id` и `name`.

Это не означает, что поле `role` удалилось:

```ts
admin.role; // доступно

const user: User = admin;
// user.role; // ошибка на уровне типа User
```

Объект в памяти остаётся тем же. Меняется только статический взгляд на него через переменную `user`.

---

## 9. Где срабатывает EPC

Проверка обычно возникает, когда свежий объектный литерал получает ожидаемый тип.

### При аннотации переменной

```ts
const user: User = {
  id: 1,
  name: 'Alex',
  age: 28, // ошибка
};
```

### При передаче аргумента

```ts
printUser({
  id: 1,
  name: 'Alex',
  age: 28, // ошибка
});
```

### При явном возвращаемом типе

```ts
function createUser(): User {
  return {
    id: 1,
    name: 'Alex',
    age: 28, // ошибка
  };
}
```

### С оператором `satisfies`

```ts
const user = {
  id: 1,
  name: 'Alex',
  age: 28, // ошибка
} satisfies User;
```

`satisfies` проверяет совместимость литерала с типом, но сохраняет более точный выведенный тип выражения.

---

## 10. Fresh object literal

Термин **fresh object literal** описывает объектный литерал, который проверяется непосредственно в контексте ожидаемого типа.

```ts
const options: RequestOptions = {
  timeout: 1000,
  retries: 3, // EPC
};
```

После сохранения без контекстной аннотации литерал получает собственный выведенный тип:

```ts
const options = {
  timeout: 1000,
  retries: 3,
};

const requestOptions: RequestOptions = options; // обычная совместимость
```

«Свежесть» — внутреннее поведение проверки типов, а не свойство объекта во время выполнения.

---

## 11. EPC не означает exact types

Тип:

```ts
type User = {
  id: number;
};
```

не означает «объект имеет ровно одно поле». Он означает «у объекта гарантированно есть `id: number`».

```ts
const admin = {
  id: 1,
  role: 'admin',
};

const user: User = admin; // корректно
```

EPC — эвристическая дополнительная диагностика для литералов, а не общее правило запрета дополнительных свойств.

Поэтому нельзя использовать TypeScript-тип как гарантию отсутствия других полей в runtime-объекте.

---

## 12. Необязательные свойства

```ts
type Config = {
  url: string;
  timeout?: number;
};
```

Оба значения допустимы:

```ts
const first: Config = { url: '/api' };
const second: Config = { url: '/api', timeout: 1000 };
```

Неизвестное поле всё равно проверяется:

```ts
const config: Config = {
  url: '/api',
  retries: 3, // ошибка EPC
};
```

При включённом `exactOptionalPropertyTypes` запись `timeout?: number` точнее отличает отсутствие свойства от присутствующего `timeout: undefined`:

```ts
const config: Config = {
  url: '/api',
  timeout: undefined,
  // При exactOptionalPropertyTypes — ошибка,
  // если undefined явно не входит в тип timeout.
};
```

---

## 13. Index signature и разрешённые дополнительные поля

Если дополнительные поля являются частью контракта, это нужно выразить явно:

```ts
type User = {
  id: number;
  name: string;
  [key: string]: unknown;
};

const user: User = {
  id: 1,
  name: 'Alex',
  department: 'Frontend',
};
```

Но широкая index signature ослабляет проверку опечаток. Лучше ограничивать значения:

```ts
type NumberDictionary = {
  [key: string]: number;
};

const metrics: NumberDictionary = {
  requests: 10,
  errors: 2,
};
```

Или отделить динамические поля:

```ts
type User = {
  id: number;
  name: string;
  metadata: Record<string, unknown>;
};
```

Последний вариант обычно лучше документирует структуру.

---

## 14. Union types и excess properties

Для union TypeScript проверяет, может ли литерал соответствовать допустимым вариантам.

```ts
type Result =
  | { status: 'success'; data: string[] }
  | { status: 'error'; error: string };

const result: Result = {
  status: 'success',
  data: [],
  error: 'Unexpected', // ошибка
};
```

Лучше использовать discriminated union с уникальным литеральным полем. Он помогает и EPC, и последующему type narrowing.

Плохая модель с необязательными полями допускает невозможные состояния:

```ts
type Result = {
  status: 'success' | 'error';
  data?: string[];
  error?: string;
};
```

Предпочтительнее явно описывать каждый вариант union.

---

## 15. Weak types и «no properties in common»

Тип, состоящий только из необязательных свойств, называют weak type:

```ts
type Options = {
  timeout?: number;
  retries?: number;
};
```

Даже промежуточная переменная может вызвать ошибку, если у неё нет ни одного общего свойства:

```ts
const value = {
  url: '/api',
};

// const options: Options = value;
// Ошибка: у типов нет общих свойств
```

Такая диагностика защищает от случайного присваивания совершенно не связанного объекта.

Если связь действительно задумана, контракт нужно сделать явнее, а не механически подавлять ошибку.

---

## 16. Объектный spread

Spread может влиять на проверку литерала:

```ts
type User = {
  id: number;
};

const extra = {
  role: 'admin',
};

const user: User = {
  id: 1,
  ...extra,
};
```

Свойства, пришедшие через spread, обычно не диагностируются как обычные явно записанные excess properties. Но прямое неизвестное поле проверяется:

```ts
const user: User = {
  id: 1,
  ...extra,
  department: 'Frontend', // ошибка EPC
};
```

Не стоит использовать spread как способ сознательно скрыть несовпадение контракта.

---

## 17. `satisfies`, аннотация и `as`

### Аннотация типа

```ts
type Theme = {
  mode: 'light' | 'dark';
};

const theme: Theme = {
  mode: 'dark',
};
```

Переменная рассматривается через тип `Theme`.

### `satisfies`

```ts
const theme = {
  mode: 'dark',
} satisfies Theme;
```

Оператор проверяет соответствие, включая лишние свойства свежего литерала, и сохраняет более точный вывод типа выражения.

```ts
const theme = {
  mode: 'dark',
  typo: true, // ошибка EPC
} satisfies Theme;
```

### Type assertion `as`

```ts
const theme = {
  mode: 'dark',
  typo: true,
} as Theme;
```

`as` просит компилятор довериться разработчику и может подавить полезную диагностику. Он не проверяет и не изменяет объект во время выполнения.

| Конструкция | Проверяет совместимость | Сохраняет точный тип выражения | Может скрыть ошибку |
|---|---:|---:|---:|
| `const x: T = ...` | Да | Обычно нет | Нет |
| `... satisfies T` | Да | Да | Нет |
| `... as T` | Ограниченно, как утверждение | Нет | Да |

Для конфигурационных объектов часто удобнее `satisfies`.

---

## 18. Как моделировать «точный» тип

TypeScript не предоставляет универсальные exact object types как базовое поведение. Для отдельных generic API можно сравнить лишние ключи:

```ts
type NoExtraProperties<Expected, Actual extends Expected> =
  Actual & Record<Exclude<keyof Actual, keyof Expected>, never>;

type User = {
  id: number;
};

function saveUser<T extends User>(
  user: NoExtraProperties<User, T>,
) {}

saveUser({ id: 1 });

saveUser({
  id: 1,
  role: 'admin', // ошибка: role должен быть never
});
```

У такого подхода есть ограничения с unions, index signatures и сложными generic-типами. Его стоит применять только там, где запрет дополнительных ключей действительно является частью API.

Для runtime-гарантии всё равно нужен валидатор, который либо отклонит, либо удалит неизвестные поля.

---

## 19. Как добавить номинальность через brand

Иногда одинаковую структуру нужно разделить по смыслу:

```ts
type UserId = string & {
  readonly __brand: 'UserId';
};

type OrderId = string & {
  readonly __brand: 'OrderId';
};
```

```ts
function createUserId(value: string): UserId {
  return value as UserId;
}

function createOrderId(value: string): OrderId {
  return value as OrderId;
}

const userId = createUserId('1');
const orderId = createOrderId('1');

// const wrong: UserId = orderId;
// Ошибка благодаря разным brand
```

Более защищённый вариант для библиотечного кода — `unique symbol`:

```ts
declare const userIdBrand: unique symbol;

type UserId = string & {
  readonly [userIdBrand]: true;
};
```

Brand существует прежде всего на уровне типов. Функция создания должна проверять значение, если у идентификатора есть runtime-ограничения.

---

## 20. Влияние generics

Generic-параметр влияет на совместимость только тогда, когда используется в структуре.

```ts
type EmptyBox<T> = {};

let stringBox: EmptyBox<string> = {};
let numberBox: EmptyBox<number> = {};

stringBox = numberBox; // структуры одинаковы
```

Если `T` используется в поле, типы различаются:

```ts
type Box<T> = {
  value: T;
};

let text: Box<string> = { value: 'hello' };
let count: Box<number> = { value: 1 };

// text = count; // ошибка
```

Имя generic-типа само по себе не создаёт номинальность.

---

## 21. `readonly` и структурная совместимость

`readonly` прежде всего запрещает запись через конкретную ссылку:

```ts
type ReadonlyPoint = {
  readonly x: number;
};

const point: ReadonlyPoint = { x: 1 };

// point.x = 2; // ошибка
```

Он не замораживает объект во время выполнения и обычно не создаёт отдельную номинальную идентичность типа. Другая изменяемая ссылка на тот же объект всё ещё может изменить значение.

```ts
const mutable = { x: 1 };
const readonlyView: ReadonlyPoint = mutable;

mutable.x = 2;
console.log(readonlyView.x); // 2
```

Для runtime-неизменяемости нужны другие механизмы, например `Object.freeze`, причём глубокая неизменяемость требует рекурсивного решения.

---

## 22. Structural typing не проверяет runtime-данные

TypeScript-типы удаляются при компиляции:

```ts
type User = {
  id: number;
  name: string;
};

const user = JSON.parse(raw) as User;
```

`as User` не проверит входные данные и не удалит лишние поля.

На границах системы нужно использовать runtime-проверку:

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

```ts
const value: unknown = JSON.parse(raw);

if (!isUser(value)) {
  throw new Error('Invalid user');
}

// value: User
```

EPC работает только во время компиляции с известным литералом. Оно не способно проверить ответ API, JSON, данные формы или содержимое хранилища.

---

## 23. Частые ошибки

### Ошибка 1. Считать, что тип запрещает любые дополнительные поля

```ts
type User = { id: number };
```

Этот тип гарантирует наличие `id`, а не точный набор ключей.

### Ошибка 2. Называть поведение с переменной багом

```ts
const value = { id: 1, extra: true };
const user: User = value;
```

Это обычная структурная совместимость. EPC специально применяется к свежим литералам.

### Ошибка 3. Обходить EPC через `as`

```ts
const config = {
  timeuot: 1000,
} as Config;
```

Так можно скрыть настоящую опечатку. Лучше исправить структуру или явно описать допустимые динамические ключи.

### Ошибка 4. Добавлять `[key: string]: any`

```ts
type Config = {
  timeout: number;
  [key: string]: any;
};
```

Это почти отключает полезную проверку полей. Если динамические значения нужны, предпочтительнее `unknown`, ограниченный union либо отдельное поле `metadata`.

### Ошибка 5. Считать `implements` номинальной связью

Другой объект без `implements` всё равно совместим, если имеет нужную публичную структуру.

### Ошибка 6. Ожидать runtime-фильтрацию полей

Аннотация типа не создаёт новый объект и не удаляет лишние свойства.

### Ошибка 7. Забывать о приватных членах классов

`private` и `protected` учитывают происхождение декларации и делают совместимость классов более номинальной.

### Ошибка 8. Пытаться получить exact type только через EPC

EPC не срабатывает одинаково для всех выражений. Если точность является частью контракта, её нужно моделировать отдельно и валидировать в runtime.

---

## 24. Вопросы для собеседования

### 1. Что такое структурная типизация?

Совместимость типов определяется набором и типами их членов. Имя типа и явная декларация наследования обычно не важны.

### 2. Как TypeScript проверяет совместимость объектов?

Исходный объект должен содержать все обязательные свойства целевого типа с совместимыми типами. Дополнительные свойства обычно разрешены.

### 3. Что такое excess property checking?

Дополнительная проверка неизвестных полей у свежих объектных литералов в контексте ожидаемого объектного типа.

### 4. Зачем нужен EPC?

Он помогает находить опечатки, неправильные настройки и неверное понимание контракта объекта.

### 5. Почему прямой литерал вызывает ошибку, а переменная — нет?

К литералу применяется EPC. Переменная уже имеет собственный выведенный тип и проверяется по общим правилам структурной совместимости.

### 6. Делает ли EPC тип точным?

Нет. Это дополнительная диагностика для определённых литеральных выражений, а не универсальный запрет лишних ключей.

### 7. Удаляются ли дополнительные свойства после присваивания узкому типу?

Нет. Типы не меняют runtime-объект. Через узкую ссылку поля не видны статически, но физически остаются в объекте.

### 8. Когда обычно срабатывает EPC?

При прямой аннотации литерала, передаче литерала аргументом, возврате литерала при известном типе результата и проверке через `satisfies`.

### 9. Как разрешить произвольные дополнительные поля?

Описать их через index signature или `Record`, желательно с ограниченным типом значения, либо вынести их в отдельное поле.

### 10. Чем `satisfies` отличается от `as`?

`satisfies` проверяет совместимость и сохраняет точный выведенный тип. `as` утверждает тип и может скрыть ошибку.

### 11. Является ли TypeScript полностью структурным?

Преимущественно да, но приватные и защищённые члены классов должны иметь общее происхождение. Branded types также позволяют искусственно добавить номинальность.

### 12. Что происходит с интерфейсами и `implements` в runtime?

Они удаляются при компиляции. `implements` только проверяет декларацию класса и не меняет объект JavaScript.

### 13. Как structural typing работает с функциями?

Сравниваются параметры и результат. Callback может игнорировать дополнительные переданные аргументы, но не должен требовать параметр, который вызывающий код не гарантирует.

### 14. Может ли generic-параметр не влиять на совместимость?

Да. Если параметр не используется в структуре, разные специализации могут иметь одинаковую форму и быть совместимыми.

### 15. Проверяет ли EPC данные API?

Нет. Это compile-time-проверка литералов. Для внешних данных требуется runtime-валидация.

### 16. Что такое weak type?

Объектный тип, состоящий только из необязательных свойств. TypeScript может отклонить присваивание объекта без единого общего поля даже через переменную.

### 17. Как запретить смешивание `UserId` и `OrderId`, если оба являются строками?

Использовать branded/opaque type, например пересечение строки с уникальным phantom-свойством.

---

## 25. Практические задачи

### Задача 1. Объясните результат

```ts
type User = {
  id: number;
};

const first: User = {
  id: 1,
  role: 'admin',
};

const data = {
  id: 1,
  role: 'admin',
};

const second: User = data;
```

<details>
<summary>Ответ</summary>

`first` вызывает ошибку excess property checking, потому что справа находится свежий объектный литерал. `second` корректен: переменная `data` содержит обязательный `id: number` и проходит обычную структурную проверку.

</details>

### Задача 2. Найдите опечатку без потери точного типа

```ts
type Config = {
  mode: 'development' | 'production';
  timeout: number;
};
```

Нужно проверить объект и сохранить литеральный тип `mode`.

<details>
<summary>Решение</summary>

```ts
const config = {
  mode: 'production',
  timeout: 1000,
} satisfies Config;
```

`satisfies` проверяет ключи и значения, не заменяя выведенный тип объекта типом `Config`.

</details>

### Задача 3. Почему классы несовместимы?

```ts
class A {
  private value = 1;
}

class B {
  private value = 1;
}

// const a: A = new B();
```

<details>
<summary>Ответ</summary>

Приватные поля имеют разное происхождение. Для `private` и `protected` одного совпадения структуры недостаточно.

</details>

### Задача 4. Разрешите metadata, сохранив безопасность

Нужно разрешить дополнительные пользовательские данные, не добавляя `[key: string]: any` ко всему объекту.

<details>
<summary>Решение</summary>

```ts
type User = {
  id: number;
  name: string;
  metadata: Record<string, unknown>;
};
```

Динамические поля изолированы, а их значения требуют narrowing перед использованием.

</details>

### Задача 5. Разделите одинаковые идентификаторы

<details>
<summary>Решение</summary>

```ts
declare const userIdBrand: unique symbol;
declare const orderIdBrand: unique symbol;

type UserId = string & {
  readonly [userIdBrand]: true;
};

type OrderId = string & {
  readonly [orderIdBrand]: true;
};
```

Несмотря на общую основу `string`, типы имеют разные структуры брендов и больше не совместимы.

</details>

---

## 26. Готовый развёрнутый ответ для интервью

> TypeScript использует преимущественно структурную типизацию: типы совместимы, когда исходное значение содержит все обязательные члены целевого типа с совместимыми типами. Поэтому объект может быть передан как интерфейс без `implements`, а дополнительные свойства переменной обычно не мешают присваиванию.
>
> Для свежих объектных литералов действует дополнительная проверка excess properties. Если напрямую присвоить или передать литерал с неизвестным полем, TypeScript покажет ошибку — это помогает находить опечатки и неверные настройки. Если сначала сохранить объект в переменную, обычно останется только структурная проверка, поэтому дополнительные поля разрешаются. EPC не делает объектные типы точными, не удаляет поля и не проверяет runtime-данные.
>
> Из важных исключений: `private` и `protected` у классов учитывают происхождение декларации, а номинальность для доменных значений можно смоделировать branded types. Для конфигураций удобно использовать `satisfies`: он проверяет соответствие и сохраняет точный вывод типа, тогда как `as` может скрыть ошибку. Внешние данные всё равно нужно валидировать во время выполнения.

---

## 27. Мини-шпаргалка

```ts
type User = {
  id: number;
  name: string;
};

// Обычная структурная совместимость
const admin = {
  id: 1,
  name: 'Alex',
  role: 'admin',
};

const user: User = admin; // OK

// Fresh object literal + EPC
const user2: User = {
  id: 1,
  name: 'Alex',
  role: 'admin', // Error
};

// Проверка с сохранением точного вывода
const user3 = {
  id: 1,
  name: 'Alex',
} satisfies User;

// Явные динамические свойства
type Entity = {
  id: number;
  metadata: Record<string, unknown>;
};

// Номинальность поверх структурной системы
declare const brand: unique symbol;
type UserId = string & { readonly [brand]: true };
```

---

## 28. Что повторить перед собеседованием

- дать определение структурной типизации;
- сравнить structural и nominal typing;
- объяснить направление `source` → `target`;
- показать совместимость объекта с дополнительными полями;
- дать определение excess property checking;
- объяснить fresh object literal;
- объяснить разницу между прямым литералом и переменной;
- назвать места, где срабатывает EPC;
- объяснить, почему EPC не является exact type;
- сравнить аннотацию, `satisfies` и `as`;
- объяснить index signatures и их риски;
- рассказать об исключении для `private`/`protected`;
- показать branded type;
- объяснить структурную совместимость функций;
- отделить compile-time-проверку от runtime-валидации.

