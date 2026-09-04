# Type Guards в TypeScript

## Короткий ответ для собеседования

**Type guard** — это проверка, которая выполняется во время работы программы и позволяет TypeScript сузить тип значения внутри определённой ветки кода.

```ts
function print(value: string | number) {
  if (typeof value === 'string') {
    console.log(value.toUpperCase()); // value: string
  } else {
    console.log(value.toFixed(2));    // value: number
  }
}
```

Встроенные type guards:

- `typeof`;
- `instanceof`;
- оператор `in`;
- сравнение значений;
- проверка на truthy/falsy;
- `Array.isArray()`;
- discriminated unions.

Для сложных типов можно создать пользовательский type guard с предикатом `value is Type`:

```ts
function isString(value: unknown): value is string {
  return typeof value === 'string';
}
```

Важно: TypeScript удаляет типы при компиляции. Type guard не возвращает удалённую информацию о типе — он проверяет реальные свойства значения, доступные в JavaScript во время выполнения.

---

## 1. Зачем нужны type guards

Если значение имеет union-тип или `unknown`, TypeScript не позволяет сразу обращаться к членам конкретного типа:

```ts
function normalize(value: string | string[]) {
  // value.toUpperCase();
  // Ошибка: метод существует не у всех членов union
}
```

Сначала нужно доказать компилятору, какой вариант находится в текущей ветке:

```ts
function normalize(value: string | string[]) {
  if (Array.isArray(value)) {
    return value.map(item => item.trim()); // string[]
  }

  return value.toUpperCase(); // string
}
```

Type guard связывает две части:

1. runtime-проверку JavaScript;
2. статический анализ TypeScript.

Сам процесс уточнения типа называется **type narrowing**.

---

## 2. Type guard не меняет значение

Проверка только уточняет знания компилятора:

```ts
let value: string | number = 'hello';

if (typeof value === 'string') {
  // В этой ветке value имеет тип string.
  // Само значение не было преобразовано.
  console.log(value.length);
}
```

После выхода из ветки тип может снова стать широким:

```ts
function example(value: string | number) {
  if (typeof value === 'string') {
    value.toUpperCase();
  }

  // Здесь по общему контракту value снова string | number.
}
```

TypeScript анализирует поток управления: `if`, `else`, ранние `return`, `throw`, присваивания и другие конструкции.

---

## 3. Guard через `typeof`

`typeof` удобно использовать для примитивов и функций:

```ts
function format(value: string | number | boolean) {
  if (typeof value === 'string') {
    return value.trim();
  }

  if (typeof value === 'number') {
    return value.toFixed(2);
  }

  return value ? 'yes' : 'no';
}
```

Результаты `typeof`, которые понимает TypeScript:

| Проверка | Сужение |
|---|---|
| `typeof value === 'string'` | `string` |
| `typeof value === 'number'` | `number` |
| `typeof value === 'boolean'` | `boolean` |
| `typeof value === 'bigint'` | `bigint` |
| `typeof value === 'symbol'` | `symbol` |
| `typeof value === 'undefined'` | `undefined` |
| `typeof value === 'function'` | функция |
| `typeof value === 'object'` | объект или `null` |

### Ловушка с `null`

```ts
typeof null; // 'object'
```

Поэтому одной проверки недостаточно:

```ts
function read(value: object | null) {
  if (typeof value === 'object' && value !== null) {
    // value: object
  }
}
```

### Ограничение `typeof`

Он не различает массив, дату и обычный объект:

```ts
typeof [];         // 'object'
typeof new Date(); // 'object'
typeof {};         // 'object'
```

Для них нужны специализированные проверки.

---

## 4. Guard через `instanceof`

`instanceof` проверяет, находится ли `Constructor.prototype` в цепочке прототипов объекта.

```ts
function formatDate(value: Date | string) {
  if (value instanceof Date) {
    return value.toISOString(); // Date
  }

  return new Date(value).toISOString(); // string
}
```

С пользовательскими классами:

```ts
class ApiError extends Error {
  constructor(
    message: string,
    readonly status: number,
  ) {
    super(message);
  }
}

function handle(error: unknown) {
  if (error instanceof ApiError) {
    console.log(error.status);
    return;
  }

  if (error instanceof Error) {
    console.log(error.message);
  }
}
```

### Ограничения `instanceof`

`interface` и `type` отсутствуют во время выполнения:

```ts
interface User {
  id: number;
}

// value instanceof User // ошибка: User — только тип
```

Объект из JSON не становится экземпляром класса:

```ts
class User {
  constructor(readonly id: number) {}
}

const value = JSON.parse('{"id": 1}');

value instanceof User; // false
```

Кроме того, `instanceof` может дать неожиданный результат для объектов из другого JavaScript realm, например другого `iframe`, потому что там используются другие объекты-конструкторы.

---

## 5. Guard через оператор `in`

Оператор `in` проверяет наличие свойства в объекте или его цепочке прототипов.

```ts
type Admin = {
  permissions: string[];
};

type Customer = {
  purchases: number;
};

function describe(user: Admin | Customer) {
  if ('permissions' in user) {
    return user.permissions.join(', '); // Admin
  }

  return user.purchases; // Customer
}
```

`in` видит как собственные, так и унаследованные свойства:

```ts
'toString' in {}; // true
```

Для проверки только собственного свойства можно использовать:

```ts
Object.hasOwn(value, 'id');
```

### Необязательные свойства

Если свойство объявлено необязательным у нескольких вариантов, оно не всегда однозначно определяет тип:

```ts
type Human = { swim?: () => void };
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Human | Fish | Bird) {
  if ('swim' in animal) {
    // Human | Fish
  } else {
    // Human | Bird
  }
}
```

Необязательный член может оказаться и в положительной, и в отрицательной ветке.

---

## 6. Equality narrowing

Строгое сравнение тоже является guard:

```ts
function compare(a: string | number, b: string | boolean) {
  if (a === b) {
    // Единственный общий тип — string.
    a.toUpperCase();
    b.toUpperCase();
  }
}
```

Проверка литерала:

```ts
type Result = 'success' | 'error';

function render(result: Result) {
  if (result === 'success') {
    // result: 'success'
  } else {
    // result: 'error'
  }
}
```

### Удаление `null` и `undefined`

Строгая версия:

```ts
if (value !== null && value !== undefined) {
  // value больше не null | undefined
}
```

Краткая версия:

```ts
if (value != null) {
  // Неравенство намеренно нестрогое:
  // исключены одновременно null и undefined.
}
```

Это один из немногих случаев, когда `!= null` может быть осознанным и полезным решением.

---

## 7. Truthiness narrowing

Условие `if (value)` исключает falsy-значения:

```ts
function print(value: string | null | undefined) {
  if (value) {
    console.log(value.toUpperCase());
  }
}
```

Falsy-значения JavaScript:

- `false`;
- `0` и `-0`;
- `0n`;
- пустая строка `''`;
- `NaN`;
- `null`;
- `undefined`.

### Частая ошибка

```ts
function printLength(value: string | null) {
  if (value) {
    console.log(value.length);
  }
}
```

Код не обрабатывает пустую строку, хотя она является допустимым `string`.

Если требуется убрать только `null`, лучше написать точную проверку:

```ts
if (value !== null) {
  console.log(value.length); // включая ''
}
```

---

## 8. `Array.isArray()`

Надёжный runtime-способ проверить массив:

```ts
function first(value: string | string[]) {
  if (Array.isArray(value)) {
    return value[0]; // string[]
  }

  return value.charAt(0); // string
}
```

В отличие от `instanceof Array`, `Array.isArray()` корректно работает с массивами из другого realm.

---

## 9. Discriminated unions

**Discriminated union** — union объектов с общим полем-дискриминатором, имеющим разные литеральные значения.

```ts
type RequestState =
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; error: Error };
```

Поле `status` служит type guard:

```ts
function render(state: RequestState): string {
  switch (state.status) {
    case 'loading':
      return 'Загрузка';

    case 'success':
      return state.data.join(', ');

    case 'error':
      return state.error.message;
  }
}
```

Это обычно надёжнее модели с набором необязательных полей, потому что union не позволяет создать противоречивое состояние.

### Exhaustive checking

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

function render(state: RequestState): string {
  switch (state.status) {
    case 'loading':
      return 'Загрузка';
    case 'success':
      return state.data.join(', ');
    case 'error':
      return state.error.message;
    default:
      return assertNever(state);
  }
}
```

Если в union добавят новый вариант, компилятор потребует обработать его.

---

## 10. Пользовательский type guard

Для сложной проверки можно написать функцию с **type predicate**:

```ts
type User = {
  id: number;
  name: string;
};

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

Использование:

```ts
const payload: unknown = JSON.parse('{"id":1,"name":"Ann"}');

if (isUser(payload)) {
  console.log(payload.name.toUpperCase()); // payload: User
}
```

Синтаксис предиката:

```ts
parameterName is Type
```

Имя слева должно совпадать с именем параметра функции:

```ts
function isNumber(value: unknown): value is number {
  return typeof value === 'number';
}
```

### Главное правило: TypeScript доверяет предикату

Компилятор не доказывает, что реализация действительно проверяет заявленный тип:

```ts
function isUser(value: unknown): value is User {
  return true; // компилируется, но это ложь
}
```

```ts
const data: unknown = null;

if (isUser(data)) {
  data.name.toUpperCase(); // runtime-ошибка
}
```

Type predicate похож на ручной контракт. Ошибка в guard делает программу типобезопасной только на бумаге.

Практические правила:

- принимать внешние данные как `unknown`, а не `any`;
- проверять каждое поле, на которое затем опирается программа;
- отдельно проверять вложенные объекты и массивы;
- тестировать положительные и отрицательные случаи;
- для больших схем использовать runtime-валидатор.

---

## 11. Проверка вложенных данных

Поверхностной проверки часто недостаточно:

```ts
type User = {
  id: number;
  profile: {
    email: string;
  };
};
```

Полный guard:

```ts
function isUser(value: unknown): value is User {
  if (typeof value !== 'object' || value === null) {
    return false;
  }

  if (!('id' in value) || typeof value.id !== 'number') {
    return false;
  }

  if (!('profile' in value)) {
    return false;
  }

  const profile = value.profile;

  return (
    typeof profile === 'object' &&
    profile !== null &&
    'email' in profile &&
    typeof profile.email === 'string'
  );
}
```

TypeScript проверяет типы исходного кода, но не гарантирует форму ответа сервера, содержимое `localStorage` или распарсенный JSON. На границах системы нужна runtime-проверка.

---

## 12. Assertion functions

Assertion function не возвращает `boolean`. Если условие неверно, она должна прервать обычное выполнение, обычно через `throw`.

### `asserts value is Type`

```ts
function assertUser(value: unknown): asserts value is User {
  if (!isUser(value)) {
    throw new TypeError('Invalid user');
  }
}
```

После успешного вызова тип сужен во внешнем потоке:

```ts
const payload: unknown = JSON.parse('{"id":1,"name":"Ann"}');

assertUser(payload);

payload.name.toUpperCase(); // payload: User
```

### `asserts condition`

```ts
function assert(
  condition: unknown,
  message: string,
): asserts condition {
  if (!condition) {
    throw new Error(message);
  }
}
```

```ts
function getLength(value: string | null) {
  assert(value !== null, 'Value is required');
  return value.length; // string
}
```

### Guard и assertion function

| Type guard | Assertion function |
|---|---|
| Возвращает `boolean` | Обычно возвращает `void` или бросает ошибку |
| Подходит для ветвления | Подходит для проверки предусловий |
| `value is User` | `asserts value is User` |
| У вызывающего кода есть ветки `true`/`false` | После успешного вызова остаётся только допустимый путь |

---

## 13. Type guard в `filter`

Частый сценарий — удалить `null` и `undefined` из массива:

```ts
function isNonNullable<T>(
  value: T,
): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const values: Array<string | null | undefined> = [
  'a',
  null,
  'b',
  undefined,
];

const strings = values.filter(isNonNullable);
// string[]
```

Не заменяйте точную проверку на `Boolean`, если falsy-значения допустимы:

```ts
const numbers = [0, 1, null].filter(Boolean);
// Во время выполнения 0 тоже будет удалён.
```

Фильтрация union:

```ts
type Doctor = {
  type: 'doctor';
  specialty: string;
};

type Nurse = {
  type: 'nurse';
  department: string;
};

type Staff = Doctor | Nurse;

function isDoctor(member: Staff): member is Doctor {
  return member.type === 'doctor';
}

const staff: Staff[] = [];
const doctors = staff.filter(isDoctor); // Doctor[]
```

Современные версии TypeScript способны вывести предикат для некоторых простых функций. Явная сигнатура всё равно полезна для публичных и сложных guards: она документирует контракт и делает намерение очевидным.

---

## 14. Универсальный guard наличия свойства

```ts
function hasProperty<K extends PropertyKey>(
  value: unknown,
  key: K,
): value is Record<K, unknown> {
  return (
    typeof value === 'object' &&
    value !== null &&
    key in value
  );
}
```

```ts
function readId(value: unknown) {
  if (hasProperty(value, 'id')) {
    // value.id: unknown
    if (typeof value.id === 'number') {
      return value.id;
    }
  }

  return null;
}
```

Guard доказывает только наличие ключа. Он не должен утверждать тип содержимого, если его не проверил.

---

## 15. Композиция guards

Небольшие guards можно переиспользовать:

```ts
type Guard<T> = (value: unknown) => value is T;

function isArrayOf<T>(
  value: unknown,
  itemGuard: Guard<T>,
): value is T[] {
  return Array.isArray(value) && value.every(itemGuard);
}

function isString(value: unknown): value is string {
  return typeof value === 'string';
}

const value: unknown = ['a', 'b'];

if (isArrayOf(value, isString)) {
  value.map(item => item.toUpperCase()); // string[]
}
```

Такой подход удобен для простых структур. Для больших моделей ручные guards быстро становятся громоздкими.

---

## 16. Guard через конструктор

```ts
type Constructor<T> = abstract new (...args: any[]) => T;

function isInstanceOf<T>(
  value: unknown,
  Constructor: Constructor<T>,
): value is T {
  return value instanceof Constructor;
}
```

```ts
if (isInstanceOf(new Date(), Date)) {
  // значение: Date
}
```

Такой helper подходит только для сущностей, у которых существует runtime-конструктор. Для `type` и `interface` он неприменим.

---

## 17. `this`-based type guards

Метод класса может уточнять тип `this`:

```ts
class FileSystemObject {
  isFile(): this is FileRepresentation {
    return this instanceof FileRepresentation;
  }
}

class FileRepresentation extends FileSystemObject {
  content = '';
}

class Directory extends FileSystemObject {
  children: FileSystemObject[] = [];
}

function read(entry: FileSystemObject) {
  if (entry.isFile()) {
    console.log(entry.content); // FileRepresentation
  }
}
```

Можно уточнить только отдельное свойство, сохранив остальную структуру объекта:

```ts
class Box<T> {
  value: T | undefined;

  hasValue(): this is this & { value: T } {
    return this.value !== undefined;
  }
}
```

```ts
const box = new Box<string>();

if (box.hasValue()) {
  box.value.toUpperCase(); // string
}
```

---

## 18. Type guard, `as` и `satisfies`

Эти механизмы решают разные задачи.

### Type guard

```ts
if (isUser(value)) {
  value.name;
}
```

- выполняет реальную runtime-проверку;
- сужает тип в потоке управления;
- может защищать границу с внешними данными.

### Type assertion `as`

```ts
const user = value as User;
```

- ничего не проверяет во время выполнения;
- сообщает компилятору: «доверься мне»;
- может скрыть ошибку.

### Оператор `satisfies`

```ts
const config = {
  mode: 'production',
} satisfies { mode: 'development' | 'production' };
```

- проверяет выражение при компиляции;
- сохраняет более точный выведенный тип;
- не валидирует неизвестные runtime-данные.

| Механизм | Runtime-проверка | Сужение потока | Основная задача |
|---|---:|---:|---|
| Type guard | Да | Да | Определить реальный вариант значения |
| `as` | Нет | Нет, это утверждение | Вручную назначить взгляд компилятора на тип |
| `satisfies` | Нет | Нет | Проверить совместимость выражения без потери вывода |

---

## 19. Type guard и runtime-валидация

Пользовательский guard может выступать простым валидатором, но это не всегда полноценная система валидации.

Guard обычно отвечает:

```ts
value is User // true или false
```

Полноценный валидатор также может:

- вернуть список ошибок по полям;
- преобразовать входные данные;
- применить значения по умолчанию;
- проверить сложные ограничения;
- построить TypeScript-тип из единой runtime-схемы.

Для маленькой модели ручной guard уместен. Для большого API-контракта схема часто снижает риск рассинхронизации между типом и проверкой.

---

## 20. Работа с `unknown`

`unknown` заставляет выполнить проверку перед использованием:

```ts
function getMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message;
  }

  if (typeof error === 'string') {
    return error;
  }

  return 'Unknown error';
}
```

Внешние данные разумно принимать как `unknown`:

```ts
async function loadUser(): Promise<User> {
  const response = await fetch('/api/user');
  const data: unknown = await response.json();

  if (!isUser(data)) {
    throw new Error('Invalid API response');
  }

  return data;
}
```

Аннотация или generic у `response.json()` сами по себе не проверяют ответ сервера.

---

## 21. Присваивания и потеря narrowing

После присваивания TypeScript пересчитывает текущий тип:

```ts
function example(value: string | number) {
  if (typeof value === 'string') {
    value = 10;
    // Теперь value: number
  }
}
```

Нужно быть особенно внимательным с изменяемыми объектами и callback-функциями: значение или свойство может измениться до момента использования. Если важно сохранить проверенный результат, удобно вынести его в локальную константу:

```ts
function process(input: { value?: string }) {
  if (input.value !== undefined) {
    const value = input.value;

    setTimeout(() => {
      console.log(value.toUpperCase());
    });
  }
}
```

---

## 22. Частые ошибки

### Ошибка 1. Использовать `as` вместо проверки

```ts
const user = JSON.parse(raw) as User;
```

Компилятор успокоен, но данные не проверены.

Правильнее:

```ts
const value: unknown = JSON.parse(raw);

if (!isUser(value)) {
  throw new Error('Invalid user');
}
```

### Ошибка 2. Проверить только существование поля

```ts
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'name' in value;
}
```

Значение `{ name: 42 }` пройдёт такую проверку. Нужно проверить и тип поля.

### Ошибка 3. Лживый предикат

```ts
function isNumber(value: unknown): value is number {
  return typeof value === 'string';
}
```

TypeScript поверит сигнатуре, а не смыслу реализации.

### Ошибка 4. Использовать `instanceof` с интерфейсом

Интерфейс удаляется при компиляции и не существует в JavaScript.

### Ошибка 5. Забыть про `null` при `typeof value === 'object'`

Всегда добавляйте `value !== null`, если `null` недопустим.

### Ошибка 6. Путать truthy с «значение существует»

`if (value)` исключит не только `null` и `undefined`, но и `0`, `''`, `false`.

### Ошибка 7. Считать `in` проверкой собственного свойства

`in` также проходит по цепочке прототипов.

### Ошибка 8. Делать одностороннюю проверку

Предикат должен корректно описывать обе ветки. Если функция возвращает `false`, TypeScript исключает заявленный тип.

Плохой пример:

```ts
function isShortString(
  value: string | number,
): value is string {
  return typeof value === 'string' && value.length < 10;
}
```

Если функция вернула `false`, значение всё ещё может быть длинной строкой, однако название предиката `value is string` обещает, что отрицательная ветка исключает все строки.

Лучше отделить определение типа от дополнительного бизнес-условия:

```ts
if (typeof value === 'string' && value.length < 10) {
  // короткая строка
}
```

---

## 23. Вопросы для собеседования

### 1. Что такое type guard?

Это runtime-проверка, которую TypeScript использует для сужения статического типа в конкретной ветке потока управления.

### 2. Type guard и type narrowing — одно и то же?

Нет. Type guard — условие или функция, которая даёт информацию о типе. Type narrowing — результат анализа, при котором широкий тип становится более узким.

### 3. Какие встроенные guards вы знаете?

`typeof`, `instanceof`, `in`, сравнения, truthiness, `Array.isArray()` и проверка поля-дискриминатора.

### 4. Что такое type predicate?

Это возвращаемый тип вида `parameter is Type`, с помощью которого пользовательская функция сообщает TypeScript, какой тип доказан при результате `true`.

### 5. Проверяет ли TypeScript корректность реализации пользовательского guard?

Не полностью. Компилятор доверяет заявленному предикату, поэтому неверная реализация может привести к runtime-ошибке.

### 6. Чем guard отличается от `as`?

Guard реально проверяет значение и сужает тип. `as` не выполняет проверку и только просит компилятор считать значение указанным типом.

### 7. Чем guard отличается от assertion function?

Guard возвращает `boolean` и используется для ветвления. Assertion function при ошибке прерывает выполнение, а после успешного вызова сужает тип в последующем коде.

### 8. Почему `typeof value === 'object'` недостаточно?

Потому что `typeof null === 'object'`, а также эта проверка не различает формы разных объектов.

### 9. Можно ли использовать `instanceof` с `interface`?

Нет. Интерфейсы существуют только во время проверки типов и удаляются из JavaScript-кода.

### 10. Почему `instanceof` не подходит для JSON?

`JSON.parse()` создаёт обычные объекты без прототипа пользовательского класса.

### 11. Что проверяет оператор `in`?

Наличие ключа в самом объекте или его цепочке прототипов. Он не проверяет тип значения свойства.

### 12. Почему `filter(Boolean)` не всегда хороший guard?

Он удаляет все falsy-значения, включая допустимые `0`, `''` и `false`, а его типовая семантика может не совпадать с требуемым результатом.

### 13. Как безопасно обработать результат API?

Получить его как `unknown`, выполнить runtime-валидацию через guard или схему и только после успешной проверки использовать как доменный тип.

### 14. Что такое discriminated union?

Union объектных типов с общим литеральным полем, по которому TypeScript однозначно определяет конкретный вариант.

### 15. Зачем используется `never` в `switch`?

Для exhaustive checking: при добавлении нового члена union необработанная ветка перестаёт иметь тип `never`, и компилятор показывает ошибку.

### 16. Может ли метод быть type guard?

Да. Метод может возвращать предикат для параметра или `this is Type` для текущего объекта.

### 17. Существует ли type guard после компиляции?

Проверяющий JavaScript-код существует. Type predicate и результат статического сужения удаляются, потому что являются частью системы типов TypeScript.

---

## 24. Практические задачи

### Задача 1. Написать guard для ошибки

Дано значение `unknown`. Нужно безопасно получить поле `message`, если оно является строкой.

<details>
<summary>Решение</summary>

```ts
type ErrorLike = {
  message: string;
};

function isErrorLike(value: unknown): value is ErrorLike {
  return (
    typeof value === 'object' &&
    value !== null &&
    'message' in value &&
    typeof value.message === 'string'
  );
}
```

</details>

### Задача 2. Отфильтровать числа

```ts
const values: unknown[] = [1, '2', null, 3];
```

Получить `number[]`.

<details>
<summary>Решение</summary>

```ts
function isNumber(value: unknown): value is number {
  return typeof value === 'number' && !Number.isNaN(value);
}

const numbers = values.filter(isNumber);
// number[]: [1, 3]
```

</details>

### Задача 3. Разобрать результат операции

```ts
type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

Написать функцию, которая возвращает данные или бросает ошибку.

<details>
<summary>Решение</summary>

```ts
function unwrap<T>(result: Result<T>): T {
  if (result.ok) {
    return result.data;
  }

  throw new Error(result.error);
}
```

Поле `ok` является дискриминатором.

</details>

### Задача 4. Найти ошибку в guard

```ts
function isUser(value: unknown): value is User {
  return typeof value === 'object';
}
```

<details>
<summary>Ответ</summary>

Проверка пропускает `null`, массивы и любые объекты без полей `id` и `name`. Предикат обещает больше, чем доказывает реализация. Нужно проверить `value !== null`, наличие всех используемых свойств и тип каждого из них.

</details>

### Задача 5. Написать assertion function

Создать функцию, после которой `string | undefined` становится `string`.

<details>
<summary>Решение</summary>

```ts
function assertDefined<T>(
  value: T,
  message = 'Value is undefined',
): asserts value is Exclude<T, undefined> {
  if (value === undefined) {
    throw new Error(message);
  }
}

let name: string | undefined;

assertDefined(name);
name.toUpperCase(); // string
```

</details>

---

## 25. Расширенный ответ для интервью

> Type guard — это runtime-проверка, результат которой TypeScript использует для сужения типа в потоке управления. Встроенными guards являются `typeof`, `instanceof`, `in`, строгие сравнения, truthiness, `Array.isArray()` и проверки дискриминирующего литерального поля в union. Для доменных типов можно написать пользовательскую функцию с предикатом `value is Type`. Если вместо `boolean` функция должна остановить выполнение при ошибке, используется assertion signature `asserts value is Type`.
>
> Важно помнить, что типы TypeScript удаляются при компиляции. Нельзя сделать `instanceof` для `interface`, а `as` не заменяет проверку данных. Пользовательскому предикату компилятор доверяет, поэтому его реализация должна действительно проверять весь заявленный контракт. Внешние данные из API, JSON или хранилища лучше принимать как `unknown` и валидировать на границе системы. Для discriminated unions полезно добавлять exhaustive checking через `never`.

---

## 26. Мини-шпаргалка

```ts
// Примитив
if (typeof value === 'string') {}

// Экземпляр класса
if (value instanceof Date) {}

// Массив
if (Array.isArray(value)) {}

// Свойство
if (typeof value === 'object' && value !== null && 'id' in value) {}

// Исключить null и undefined
if (value != null) {}

// Discriminated union
if (result.status === 'success') {}

// Пользовательский guard
function isUser(value: unknown): value is User {
  return /* настоящая проверка */;
}

// Assertion function
function assertUser(value: unknown): asserts value is User {
  if (!isUser(value)) throw new Error('Invalid user');
}

// Удалить null и undefined из массива
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

// Exhaustive checking
function assertNever(value: never): never {
  throw new Error('Unexpected value');
}
```

---

## 27. Что повторить перед собеседованием

- определить type guard и type narrowing;
- назвать встроенные способы сужения;
- объяснить ловушку `typeof null`;
- объяснить ограничения `instanceof`;
- написать предикат `value is Type` для `unknown`;
- объяснить, почему компилятор доверяет пользовательскому guard;
- отличить guard от `as`, `satisfies` и assertion function;
- отфильтровать nullable-значения с правильным итоговым типом;
- смоделировать состояние через discriminated union;
- реализовать exhaustive checking через `never`;
- объяснить необходимость runtime-валидации внешних данных.

