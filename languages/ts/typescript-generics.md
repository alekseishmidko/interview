# Generics в TypeScript — шпаргалка для собеседования

## Краткий ответ

**Generics** — механизм параметризации типов. Он позволяет написать функцию, тип, интерфейс или класс один раз и использовать его с разными типами, сохраняя связь между входными и выходными данными.

```ts
function identity<T>(value: T): T {
  return value;
}

const number = identity(42);      // number
const text = identity('hello');   // string
const user = identity({ id: 1 }); // { id: number }
```

`T` — параметр типа. При вызове TypeScript обычно сам выводит конкретный тип аргумента и подставляет его вместо `T`.

Главное отличие от `any`: generic не теряет информацию о типе.

```ts
function identityAny(value: any): any {
  return value;
}

const result = identityAny('hello');
// result: any — связь потеряна
```

Краткая формулировка для интервью:

> Generics позволяют создавать переиспользуемый типобезопасный код. Параметр типа выступает переменной на уровне типов: он принимает конкретный тип при использовании и связывает несколько позиций сигнатуры, например аргумент и результат функции.

---

## 1. Зачем нужны generics

Представим функцию, возвращающую переданное значение.

### Отдельная функция для каждого типа

```ts
function identityNumber(value: number): number {
  return value;
}

function identityString(value: string): string {
  return value;
}
```

Код дублируется.

### Через `any`

```ts
function identity(value: any): any {
  return value;
}
```

Переиспользование появилось, но типобезопасность потеряна:

```ts
const value = identity('hello');
value.notExistingMethod(); // TypeScript не остановит
```

### Через generic

```ts
function identity<T>(value: T): T {
  return value;
}
```

Теперь функция переиспользуется и сохраняет тип:

```ts
const value = identity('hello');
// value: string

// value.notExistingMethod(); // ошибка TypeScript
```

---

## 2. Базовый синтаксис

```ts
function functionName<TypeParameter>(
  argument: TypeParameter,
): TypeParameter {
  return argument;
}
```

Обычно короткие параметры называют:

- `T` — Type;
- `K` — Key;
- `V` — Value;
- `E` — Element или Error;
- `R` — Result или Return;
- `P` — Props или Parameters.

В публичном и сложном API понятное имя часто лучше одной буквы:

```ts
function wrap<Value>(value: Value): { value: Value } {
  return { value };
}
```

---

## 3. Явная передача типа и type inference

### Явный generic-аргумент

```ts
const result = identity<string>('hello');
```

### Автоматический вывод

```ts
const result = identity('hello');
```

TypeScript видит строковый аргумент и выводит `T` автоматически.

Обычно следует полагаться на inference, если он даёт правильный и понятный тип. Явный аргумент полезен, когда:

- TypeScript не может вывести тип;
- нужно задать более широкий тип;
- входных данных недостаточно;
- вывод идёт не из той позиции;
- явное указание улучшает читаемость.

```ts
const users = createStore<User>();
```

Здесь передать `User` необходимо, если функция вызывается без значения, из которого его можно вывести.

---

## 4. Generic сохраняет связь типов

Главная ценность generics — не просто «принимать разные типы», а связывать их.

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
```

```ts
const number = first([1, 2, 3]);
// number | undefined

const name = first(['Alex', 'Max']);
// string | undefined
```

Тип результата зависит от типа элементов аргумента.

Если написать union:

```ts
function first(
  items: string[] | number[],
): string | number | undefined {
  return items[0];
}
```

Результат всегда будет широким `string | number | undefined`, даже если передан `string[]`. Generic сохраняет точную зависимость.

---

## 5. Generic vs `any`

```ts
function getFirstAny(items: any[]): any {
  return items[0];
}

function getFirst<T>(items: T[]): T | undefined {
  return items[0];
}
```

| `any` | Generic |
| --- | --- |
| Отключает значительную часть проверок | Сохраняет проверку |
| Теряет тип результата | Связывает результат с аргументом |
| Разрешает неизвестные операции | Разрешает только операции, доступные для `T` |
| Ошибки проявляются в runtime | Ошибки чаще обнаруживаются компилятором |

Generic не является «улучшенным `any`». Это параметр неизвестного, но согласованного типа.

---

## 6. Generic vs `unknown`

`unknown` описывает неизвестное значение, которое нужно проверить перед использованием:

```ts
function log(value: unknown) {
  if (typeof value === 'string') {
    console.log(value.toUpperCase());
  }
}
```

Generic сохраняет конкретный тип для вызывающего кода:

```ts
function wrap<T>(value: T): { value: T } {
  return { value };
}

const wrapped = wrap('hello');
// { value: string }
```

Коротко:

- `unknown` — «тип значения неизвестен, сначала проверь»;
- generic — «конкретный тип определит вызывающий код, а API сохранит связи с ним».

---

## 7. Generic vs union

Union перечисляет допустимые варианты:

```ts
function stringify(value: string | number): string {
  return String(value);
}
```

Generic сохраняет конкретный тип:

```ts
function echo<T>(value: T): T {
  return value;
}
```

Используйте union, когда:

- логика обрабатывает конечный набор вариантов;
- между аргументом и результатом не нужно сохранять конкретную связь;
- внутри функции требуется narrowing вариантов.

Используйте generic, когда:

- результат зависит от входного типа;
- несколько аргументов должны быть связаны;
- создаётся контейнер или переиспользуемый алгоритм;
- набор допустимых типов открыт.

---

## 8. Ограничения generics — constraints

Без ограничения про `T` ничего не известно:

```ts
function getLength<T>(value: T) {
  // return value.length; // ошибка
}
```

Не у каждого типа есть `length`.

Добавим constraint:

```ts
type WithLength = {
  length: number;
};

function getLength<T extends WithLength>(value: T): number {
  return value.length;
}
```

```ts
getLength('hello');       // 5
getLength([1, 2, 3]);     // 3
getLength({ length: 10 }); // 10

// getLength(42); // ошибка
```

`T extends WithLength` означает: `T` может быть любым типом, совместимым с `WithLength`.

Constraint ограничивает допустимые типы и разрешает безопасно использовать известные члены внутри реализации.

---

## 9. Почему не нужно заменять generic самим constraint

```ts
function identityWithLength(
  value: WithLength,
): WithLength {
  return value;
}
```

Такой вариант теряет дополнительные свойства:

```ts
const result = identityWithLength({
  length: 5,
  name: 'collection',
});

// result.name // ошибка: тип результата только WithLength
```

Generic сохраняет полный конкретный тип:

```ts
function identityWithLength<T extends WithLength>(
  value: T,
): T {
  return value;
}

const result = identityWithLength({
  length: 5,
  name: 'collection',
});

result.name; // допустимо
```

Constraint задаёт минимальные требования, но `T` сохраняет дополнительные свойства.

---

## 10. Несколько параметров типа

```ts
function createPair<Key, Value>(
  key: Key,
  value: Value,
): [Key, Value] {
  return [key, value];
}
```

```ts
const pair = createPair('user', { id: 1 });
// [string, { id: number }]
```

Параметры могут зависеть друг от друга:

```ts
function merge<
  First extends object,
  Second extends object,
>(first: First, second: Second): First & Second {
  return { ...first, ...second };
}
```

```ts
const result = merge(
  { id: 1 },
  { name: 'Alex' },
);

// result: { id: number } & { name: string }
```

При конфликтующих ключах intersection может давать неудобные или невозможные типы. Реальный `Object.assign` и spread имеют собственную семантику перезаписи, поэтому сложные merge-функции требуют более точной модели.

---

## 11. Связь через `keyof`

Классический вопрос на интервью — типобезопасное получение свойства.

```ts
function getProperty<
  ObjectType,
  Key extends keyof ObjectType,
>(object: ObjectType, key: Key): ObjectType[Key] {
  return object[key];
}
```

```ts
const user = {
  id: 1,
  name: 'Alex',
  active: true,
};

const name = getProperty(user, 'name');
// string

const active = getProperty(user, 'active');
// boolean

// getProperty(user, 'email'); // ошибка
```

Здесь:

- `ObjectType` — тип объекта;
- `keyof ObjectType` — union его ключей;
- `Key` — конкретный выбранный ключ;
- `ObjectType[Key]` — тип значения по этому ключу.

---

## 12. Типобезопасное изменение свойства

```ts
function setProperty<
  ObjectType,
  Key extends keyof ObjectType,
>(
  object: ObjectType,
  key: Key,
  value: ObjectType[Key],
): void {
  object[key] = value;
}
```

```ts
setProperty(user, 'name', 'Max'); // допустимо
setProperty(user, 'active', false); // допустимо

// setProperty(user, 'active', 'yes'); // ошибка
```

Generic связывает выбранный ключ с допустимым типом значения.

---

## 13. Значение типа по умолчанию

Generic-параметру можно задать default:

```ts
type ApiResponse<Data = unknown> = {
  data: Data;
  status: number;
};
```

```ts
type UnknownResponse = ApiResponse;
// { data: unknown; status: number }

type UserResponse = ApiResponse<User>;
// { data: User; status: number }
```

Несколько параметров:

```ts
type Result<
  Data,
  ErrorType = Error,
> =
  | { success: true; data: Data }
  | { success: false; error: ErrorType };
```

Правила:

- параметр с default становится необязательным;
- обязательные generic-параметры не должны идти после необязательных;
- явно переданный тип должен соответствовать constraint;
- если inference не выбрал тип, используется default.

---

## 14. Generic type alias

```ts
type Box<T> = {
  value: T;
};

type StringBox = Box<string>;
type UserBox = Box<User>;
```

Более практичный контейнер:

```ts
type ApiResponse<Data, Meta = undefined> = {
  data: Data;
  meta: Meta;
  status: number;
};

type UsersResponse = ApiResponse<
  User[],
  { total: number; page: number }
>;
```

---

## 15. Generic interface

```ts
interface Repository<Entity, Id = string> {
  findById(id: Id): Promise<Entity | null>;
  findAll(): Promise<Entity[]>;
  save(entity: Entity): Promise<Entity>;
  delete(id: Id): Promise<void>;
}
```

```ts
class UserRepository
  implements Repository<User, string> {
  async findById(id: string) {
    return null;
  }

  async findAll() {
    return [];
  }

  async save(user: User) {
    return user;
  }

  async delete(id: string) {
    // ...
  }
}
```

Generic interface задаёт контракт, который можно специализировать для разных сущностей.

---

## 16. Generic class

```ts
class Store<State> {
  constructor(private state: State) {}

  getState(): State {
    return this.state;
  }

  setState(nextState: State): void {
    this.state = nextState;
  }
}
```

```ts
const userStore = new Store({
  name: 'Alex',
  authenticated: true,
});

userStore.setState({
  name: 'Max',
  authenticated: false,
});
```

TypeScript выводит тип `State` из аргумента конструктора.

### Статические члены

Статические члены класса не могут использовать параметр типа экземпляра:

```ts
class Container<T> {
  // static defaultValue: T; // ошибка
}
```

Причина: у класса один общий runtime-конструктор и один набор статических членов, а экземпляры могут иметь разные `T`.

---

## 17. Generic function type

Generic может принадлежать самой функции:

```ts
type Identity = <T>(value: T) => T;

const identity: Identity = value => value;
```

Здесь каждый вызов способен выбрать собственный `T`:

```ts
identity(10);      // number
identity('hello'); // string
```

Другой вариант — generic у внешнего типа:

```ts
type Mapper<Input, Output> = (
  value: Input,
) => Output;

const toLength: Mapper<string, number> =
  value => value.length;
```

Различие:

- `<T>(value: T) => T` — вызывающий код выбирает `T` при каждом вызове;
- `Mapper<string, number>` — типы фиксируются при создании специализации.

---

## 18. Generic callback

```ts
function map<Input, Output>(
  items: Input[],
  callback: (
    item: Input,
    index: number,
  ) => Output,
): Output[] {
  return items.map(callback);
}
```

```ts
const lengths = map(
  ['one', 'three'],
  word => word.length,
);
// number[]
```

TypeScript выводит:

- `Input` как `string` из массива;
- `Output` как `number` из результата callback;
- итог как `number[]`.

---

## 19. Generic factory

```ts
type Constructor<T, Args extends unknown[] = []> =
  new (...args: Args) => T;

function createInstance<
  Instance,
  Args extends unknown[],
>(
  Constructor: new (...args: Args) => Instance,
  ...args: Args
): Instance {
  return new Constructor(...args);
}
```

```ts
class User {
  constructor(
    public id: string,
    public name: string,
  ) {}
}

const user = createInstance(User, '1', 'Alex');
// User
```

Параметры конструктора и тип экземпляра выводятся автоматически.

---

## 20. Generic и utility types

Все основные utility types являются generic:

```ts
Partial<User>
Pick<User, 'id' | 'name'>
Omit<User, 'passwordHash'>
Record<Role, Permission[]>
ReturnType<typeof createUser>
Awaited<Promise<User>>
```

Упрощённая реализация `Partial`:

```ts
type MyPartial<T> = {
  [Key in keyof T]?: T[Key];
};
```

Generic-параметр `T` позволяет применить одно преобразование к любой объектной модели.

---

## 21. Generic и conditional types

Conditional type вычисляет результат на основе generic-параметра:

```ts
type ApiResult<T> =
  T extends Error
    ? { success: false; error: T }
    : { success: true; data: T };
```

```ts
type Success = ApiResult<User>;
type Failure = ApiResult<TypeError>;
```

Другой пример:

```ts
type ResultByFlag<T extends boolean> =
  T extends true ? User[] : User;
```

Conditional types помогают выразить зависимость результата от переданного generic-типа.

---

## 22. Generic и `infer`

Generic-параметр объявляется заранее:

```ts
type Box<T> = {
  value: T;
};
```

`infer` извлекает параметр из уже полученного типа:

```ts
type Unbox<T> =
  T extends Box<infer Value>
    ? Value
    : never;
```

```ts
type Result = Unbox<Box<string>>;
// string
```

Коротко:

- generic — входная переменная типа;
- `infer` — локальная переменная, выведенная при сопоставлении структуры conditional type.

---

## 23. `NoInfer<T>`

Обычно TypeScript выводит generic из всех подходящих аргументов. Иногда один аргумент должен только проверяться, но не влиять на inference.

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

`C` выводится из `colors`. `defaultColor` проверяется на соответствие полученному union, но не расширяет его.

Не путать `NoInfer<T>` с ключевым словом `infer`.

---

## 24. Const type parameters

`const` перед generic-параметром помогает сохранять более точные литеральные типы при inference.

```ts
function defineRoutes<
  const Routes extends Record<string, string>,
>(routes: Routes): Routes {
  return routes;
}
```

```ts
const routes = defineRoutes({
  home: '/',
  users: '/users',
});
```

TypeScript старается сохранить конкретные ключи и литеральные значения, похожим образом с `as const`, когда это возможно в рамках constraint.

Const type parameter:

- влияет на inference;
- не делает runtime-объект неизменяемым;
- не заменяет проверку данных;
- особенно полезен в библиотечных builder/define API.

---

## 25. Generic в React-компонентах

Generic-компонент полезен, когда props связаны с типом данных.

```tsx
type ListProps<Item> = {
  items: Item[];
  getKey: (item: Item) => React.Key;
  renderItem: (item: Item) => React.ReactNode;
};

function List<Item>({
  items,
  getKey,
  renderItem,
}: ListProps<Item>) {
  return (
    <ul>
      {items.map(item => (
        <li key={getKey(item)}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
<List
  items={users}
  getKey={user => user.id}
  renderItem={user => user.name}
/>
```

`Item` выводится из `users`, после чего callbacks получают корректно типизированного пользователя.

### Стрелочная generic-функция в TSX

Запись `<T>` может быть воспринята как JSX-тег. Используют запятую:

```tsx
const identity = <T,>(value: T): T => value;
```

Или constraint:

```tsx
const identity = <T extends unknown>(value: T): T => value;
```

---

## 26. Generic React hook

```tsx
function usePrevious<Value>(value: Value): Value | undefined {
  const ref = React.useRef<Value>(undefined);

  React.useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}
```

```tsx
const previousCount = usePrevious(count);
// number | undefined

const previousUser = usePrevious(user);
// User | undefined
```

Один hook сохраняет тип переданного значения.

---

## 27. Generic API response

```ts
interface ApiResponse<Data, Meta = undefined> {
  data: Data;
  meta: Meta;
  status: number;
}

interface PaginationMeta {
  page: number;
  total: number;
}

type UsersResponse = ApiResponse<
  User[],
  PaginationMeta
>;
```

Важно: generic-аннотация не валидирует реальный JSON.

```ts
async function request<Data>(
  url: string,
): Promise<Data> {
  const response = await fetch(url);
  return response.json();
}
```

Вызов `request<User[]>('/users')` является обещанием со стороны разработчика, а не runtime-проверкой сервера. На границе внешних данных нужен валидатор или type guard.

---

## 28. Generic repository: польза и границы

```ts
interface Entity<Id = string> {
  id: Id;
}

interface Repository<
  Model extends Entity<Id>,
  Id = string,
> {
  findById(id: Id): Promise<Model | null>;
  save(model: Model): Promise<Model>;
}
```

Generic repository полезен для общих операций, но не следует прятать за ним все различия доменов. Например, `UserRepository` и `OrderRepository` могут иметь разные запросы, транзакционные правила и ограничения.

Хороший generic выражает реальную общую связь, а не искусственно делает разные сущности одинаковыми.

---

## 29. Generic и перегрузки

Generic заменяет перегрузки, когда между входом и выходом есть единое правило:

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
```

Не нужны отдельные перегрузки для `string[]`, `number[]`, `User[]`.

Перегрузки могут быть понятнее, если поведение имеет несколько конкретных форм:

```ts
function parse(value: string): object;
function parse(value: Buffer): object;
```

Conditional generic иногда делает сигнатуру точнее, но сложнее:

```ts
type ParseResult<T> =
  T extends 'json' ? object : string;
```

Выбор зависит от читаемости публичного API.

---

## 30. Generic и narrowing

Сужение generic-параметров бывает менее очевидным, чем с обычным union.

```ts
function format<T extends string | number>(value: T) {
  if (typeof value === 'string') {
    return value.toUpperCase();
  }

  return value.toFixed(2);
}
```

Значение сужается, но компилятору иногда трудно доказать связь conditional return type с конкретной веткой реализации:

```ts
type FormatResult<T> =
  T extends string ? string : number;
```

Для сложных случаев можно использовать:

- overload signatures;
- discriminated unions;
- отдельный type guard;
- локальную внутреннюю реализацию с более широким типом;
- хорошо обоснованный assertion на границе реализации.

Не следует автоматически лечить каждую проблему generic narrowing через `as any`.

---

## 31. Variance — продвинутая тема

Variance описывает, как совместимость `Container<A>` и `Container<B>` зависит от совместимости `A` и `B`.

Пусть `Dog extends Animal`.

### Ковариантность

Тип используется только как результат:

```ts
type Producer<T> = () => T;
```

`Producer<Dog>` можно использовать там, где ожидается `Producer<Animal>`, потому что функция, возвращающая собаку, всегда возвращает животное.

### Контравариантность

Тип используется как параметр функции:

```ts
type Consumer<T> = (value: T) => void;
```

Функция, способная принять любое `Animal`, подходит там, где будут передавать только `Dog`. Проверка зависит от `strictFunctionTypes` и формы объявления метода/callback.

### Инвариантность

Тип используется и на входе, и на выходе:

```ts
type Box<T> = {
  get(): T;
  set(value: T): void;
};
```

Безопасная взаимозаменяемость становится ограниченнее.

TypeScript в основном выводит variance структурно. В продвинутых библиотечных типах доступны variance annotations `in` и `out`, но применять их следует только при точном понимании модели.

---

## 32. Generics отсутствуют в runtime

```ts
function identity<T>(value: T): T {
  return value;
}
```

После компиляции получится обычная JavaScript-функция:

```js
function identity(value) {
  return value;
}
```

Нельзя проверить параметр типа так:

```ts
function isType<T>(value: unknown) {
  // return value instanceof T; // невозможно
}
```

`T` не является runtime-значением. Для проверки нужны:

- конструктор;
- type guard;
- функция-предикат;
- схема runtime-валидации;
- дискриминатор данных.

Передача конструктора:

```ts
function isInstance<T>(
  value: unknown,
  Constructor: new (...args: any[]) => T,
): value is T {
  return value instanceof Constructor;
}
```

---

## 33. Когда generics не нужны

Generic должен связывать типы или создавать полезную абстракцию.

Бесполезный generic:

```ts
function log<T>(value: T): void {
  console.log(value);
}
```

Если `T` больше нигде не используется, достаточно:

```ts
function log(value: unknown): void {
  console.log(value);
}
```

Другой лишний generic:

```ts
function getName<T extends { name: string }>(
  value: T,
): string {
  return value.name;
}
```

Если дополнительные свойства нигде не сохраняются в типе результата, проще:

```ts
function getName(value: { name: string }): string {
  return value.name;
}
```

Практическое правило:

> Параметр типа обычно должен связывать минимум две позиции либо использоваться для вывода производного типа.

---

## 34. Частые ошибки

### Использовать `any` вместо параметра типа

Это разрывает связь входа и результата.

### Добавлять generic, который используется один раз

Если параметр типа не связывает позиции, часто достаточно конкретного типа или `unknown`.

### Слишком широкий constraint

```ts
function process<T extends object>(value: T) {
  // О свойствах value почти ничего неизвестно.
}
```

Constraint должен описывать реальные минимальные требования.

### Слишком узкий constraint

Он запрещает корректные сценарии и уменьшает переиспользование.

### Явно передавать тип без необходимости

```ts
identity<string>('hello');
```

Не ошибка, но inference уже знает тип. Избыточные аргументы увеличивают шум.

### Лгать о типе ответа API

```ts
request<User>('/user');
```

Generic не проверяет серверный JSON.

### Требовать вернуть произвольный `T`

```ts
function create<T>(): T {
  return {} as T;
}
```

Функция обещает создать любой тип, выбранный вызывающим кодом, но не располагает данными для этого. Assertion скрывает некорректный контракт.

### Забывать `undefined` при индексном доступе

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
```

Пустой массив не содержит элемента. Тип результата должен это отражать.

### Слишком много параметров типа

Большое количество взаимосвязанных generics делает API сложным. Иногда лучше ввести именованный объектный тип или разделить функцию.

### Использовать generic только ради «универсальности»

Абстракция должна соответствовать реальному общему поведению, а не скрывать различия домена.

---

## 35. Популярные вопросы на интервью

### Что такое generics?

Параметры типов, позволяющие создавать переиспользуемые типобезопасные функции, классы, интерфейсы и type aliases с сохранением связей между типами.

### Чем generic отличается от `any`?

`any` теряет информацию и отключает проверки. Generic принимает неизвестный конкретный тип и последовательно использует его во всех связанных позициях.

### Чем generic отличается от `unknown`?

`unknown` требует runtime-сужения перед использованием. Generic сохраняет тип, выбранный или выведенный для конкретного вызова.

### Чем generic отличается от union?

Union перечисляет варианты и обычно требует narrowing. Generic сохраняет конкретный тип и связь входа с результатом.

### Что такое generic constraint?

Минимальное требование к параметру типа:

```ts
T extends { id: string }
```

Оно ограничивает допустимые типы и открывает доступ к известным свойствам внутри реализации.

### Как связать ключ объекта с типом значения?

```ts
function get<T, K extends keyof T>(
  object: T,
  key: K,
): T[K] {
  return object[key];
}
```

### Что такое type inference?

Автоматический вывод generic-аргумента по значениям и контексту вызова.

### Когда тип нужно передать явно?

Когда данных для inference недостаточно, нужно выбрать более широкий тип или компилятор выводит не тот контракт, который требуется API.

### Можно ли задать generic default?

Да:

```ts
type Response<T = unknown> = {
  data: T;
};
```

### Можно ли использовать несколько параметров?

Да, и они могут зависеть друг от друга, например `K extends keyof T`.

### Существуют ли generics в JavaScript runtime?

Нет. TypeScript стирает их при компиляции.

### Можно ли сделать `instanceof T`?

Нет, потому что `T` отсутствует в runtime. Нужно передать реальный конструктор.

### Зачем нужен `NoInfer<T>`?

Чтобы аргумент проверялся против уже выведенного типа, но сам не участвовал в его inference.

### Что такое const type parameter?

Модификатор generic-параметра, предлагающий TypeScript выводить более точные литеральные типы из выражения.

### Почему generic иногда не сужается как ожидается?

Компилятор должен сохранять параметрическую связь для любого допустимого `T`. Для сложных зависимостей могут понадобиться overload, conditional type или другая модель.

### Когда generic считается лишним?

Когда параметр используется только один раз и не связывает вход, результат или другие части типа.

---

## 36. Практические задачи с решениями

### Задача 1: типобезопасный `first`

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
```

### Задача 2: универсальный `wrap`

```ts
function wrap<T>(value: T): { value: T } {
  return { value };
}
```

### Задача 3: только значения с `length`

```ts
function getLength<
  T extends { length: number },
>(value: T): number {
  return value.length;
}
```

### Задача 4: безопасное получение поля

```ts
function getProperty<
  T,
  K extends keyof T,
>(object: T, key: K): T[K] {
  return object[key];
}
```

### Задача 5: безопасное изменение поля

```ts
function setProperty<
  T,
  K extends keyof T,
>(object: T, key: K, value: T[K]): void {
  object[key] = value;
}
```

### Задача 6: generic response

```ts
interface ApiResponse<Data, Meta = undefined> {
  data: Data;
  meta: Meta;
  status: number;
}
```

### Задача 7: универсальный `map`

```ts
function map<Input, Output>(
  items: Input[],
  callback: (item: Input) => Output,
): Output[] {
  return items.map(callback);
}
```

### Задача 8: получить значение по ключу или fallback

```ts
function getOrDefault<
  T,
  K extends keyof T,
>(
  object: T,
  key: K,
  fallback: NonNullable<T[K]>,
): NonNullable<T[K]> {
  const value = object[key];

  return value == null
    ? fallback
    : (value as NonNullable<T[K]>);
}
```

Assertion здесь локализует ограничение control flow analysis для индексированного generic-значения. Контракт возвращает значение без `null | undefined`.

### Задача 9: фабрика экземпляров

```ts
function create<
  Instance,
  Args extends unknown[],
>(
  Constructor: new (...args: Args) => Instance,
  ...args: Args
): Instance {
  return new Constructor(...args);
}
```

### Задача 10: generic React-список

```tsx
type ListProps<Item> = {
  items: Item[];
  renderItem: (item: Item) => React.ReactNode;
};

function List<Item>({
  items,
  renderItem,
}: ListProps<Item>) {
  return <>{items.map(renderItem)}</>;
}
```

В реальном React-списке каждому элементу нужен стабильный `key`; его можно получить отдельным `getKey` callback.

---

## 37. Развёрнутый ответ для собеседования

> Generics — это параметры типов. Они позволяют написать одну функцию, структуру или класс для разных типов, не теряя типобезопасность. Например, `function identity<T>(value: T): T` связывает тип аргумента с типом результата: для строки вернётся строка, для пользователя — пользователь. В отличие от `any`, информация о конкретном типе сохраняется.
>
> TypeScript обычно выводит generic-аргументы автоматически. Если внутри реализации нужны определённые свойства, параметр ограничивают через constraint: `T extends { id: string }`. Несколько параметров можно связать, например `K extends keyof T`, а тип значения получить как `T[K]`. Generics поддерживаются функциями, типами, интерфейсами и классами, могут иметь default-значения и участвовать в conditional и mapped types.
>
> Generics существуют только на этапе компиляции и не валидируют runtime-данные. Поэтому нельзя сделать `instanceof T`, а `request<User>()` сам по себе не проверяет JSON от сервера. Хороший generic выражает реальную связь между несколькими позициями типа; если параметр используется только один раз, часто проще применить конкретный тип или `unknown`.

---

## 38. Мини-шпаргалка

```ts
// Identity
function identity<T>(value: T): T {
  return value;
}

// Constraint
function lengthOf<T extends { length: number }>(
  value: T,
): number {
  return value.length;
}

// Несколько параметров
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value];
}

// keyof-связь
function get<T, K extends keyof T>(
  object: T,
  key: K,
): T[K] {
  return object[key];
}

// Default
type Response<T = unknown> = {
  data: T;
};

// Generic interface
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
}

// Conditional generic
type Unwrap<T> =
  T extends PromiseLike<infer U> ? U : T;

// Не выводить тип из позиции
function choose<T>(
  values: T[],
  fallback: NoInfer<T>,
): T {
  return values[0] ?? fallback;
}

// TSX-стрелка
const wrap = <T,>(value: T) => ({ value });
```

---

## 39. Чек-лист перед интервью

- Объяснить generics без фразы «это просто `any`».
- Показать сохранение связи аргумента и результата.
- Отличать generic от `any`, `unknown` и union.
- Объяснить type inference.
- Использовать generic constraint.
- Показать, почему `T extends Constraint` сохраняет дополнительные свойства.
- Написать `getProperty<T, K extends keyof T>`.
- Использовать несколько связанных параметров.
- Задать default generic-параметра.
- Описать generic type, interface и class.
- Типизировать callback с `Input` и `Output`.
- Объяснить связь с utility и conditional types.
- Отличать generic-параметр от `infer`.
- Объяснить `NoInfer` и const type parameters.
- Знать синтаксис generic-стрелки в `.tsx`.
- Понимать основы variance.
- Помнить, что generics отсутствуют в runtime.
- Не использовать generic, если он не связывает типы.
