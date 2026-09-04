# Conditional types в TypeScript — шпаргалка для собеседования

## Краткий ответ

**Conditional types** — условные типы TypeScript, которые выбирают один из двух типов в зависимости от того, совместим ли исходный тип с заданным условием.

Синтаксис похож на тернарный оператор:

```ts
type Result<T> = T extends Condition
  ? TrueType
  : FalseType;
```

Пример:

```ts
type IsString<T> = T extends string
  ? true
  : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

Здесь `extends` означает не наследование класса, а проверку совместимости типов: можно ли считать `T` подтипом или допустимым значением `Condition`.

Условные типы используются для:

- выбора типа на основании другого типа;
- фильтрации union;
- извлечения вложенных типов через `infer`;
- рекурсивного преобразования;
- реализации utility types вроде `Exclude`, `Extract`, `NonNullable`, `ReturnType` и `Awaited`.

---

## 1. Базовый синтаксис

```ts
type Conditional<T> =
  T extends SomeType
    ? TypeIfTrue
    : TypeIfFalse;
```

Читается так:

> Если `T` совместим с `SomeType`, вернуть `TypeIfTrue`, иначе вернуть `TypeIfFalse`.

```ts
type ToLabel<T> =
  T extends string
    ? { text: T }
    : { value: T };

type StringLabel = ToLabel<string>;
// { text: string }

type NumberLabel = ToLabel<number>;
// { value: number }
```

Conditional type существует только на уровне типов и полностью удаляется при компиляции.

---

## 2. Что означает `extends` в conditional type

В этом контексте `extends` проверяет assignability — совместимость типов.

```ts
type Check<T> = T extends { id: number }
  ? 'has id'
  : 'no id';
```

```ts
type A = Check<{ id: number; name: string }>;
// 'has id'

type B = Check<{ name: string }>;
// 'no id'
```

TypeScript использует структурную типизацию. Дополнительное поле `name` не мешает совместимости с `{ id: number }`.

Это не runtime-проверка:

```ts
type Result = Check<{ id: number }>;
```

Компилятор вычисляет `Result`, но JavaScript-код для проверки объекта не генерируется.

---

## 3. Простой выбор типа

```ts
type ApiResult<T> = T extends Error
  ? { success: false; error: T }
  : { success: true; data: T };
```

```ts
type Success = ApiResult<User>;
// { success: true; data: User }

type Failure = ApiResult<TypeError>;
// { success: false; error: TypeError }
```

Conditional type строит новый тип на основе входного.

---

## 4. Conditional type vs условие в JavaScript

JavaScript-условие работает со значениями в runtime:

```ts
const result = value > 0
  ? 'positive'
  : 'negative';
```

Conditional type работает с типами во время компиляции:

```ts
type Result<T> = T extends number
  ? 'numeric type'
  : 'other type';
```

| JavaScript condition | TypeScript conditional type |
| --- | --- |
| Проверяет значение | Проверяет совместимость типов |
| Работает в runtime | Работает при компиляции |
| Остаётся в JavaScript | Удаляется из JavaScript |
| Возвращает значение | Возвращает тип |

---

## 5. Generic constraints vs conditional types

Эти конструкции используют `extends`, но решают разные задачи.

### Generic constraint ограничивает вход

```ts
type EntityId<T extends { id: unknown }> = T['id'];

type A = EntityId<{ id: number }>;
// number

// type B = EntityId<string>; // ошибка
```

`T` обязан соответствовать ограничению. Неподходящий тип нельзя передать.

### Conditional type принимает вход и выбирает результат

```ts
type EntityId<T> =
  T extends { id: infer Id }
    ? Id
    : never;

type A = EntityId<{ id: number }>;
// number

type B = EntityId<string>;
// never
```

Здесь `string` передать можно, но результатом будет false-ветка.

Коротко:

- constraint запрещает неподходящий вход;
- conditional type обрабатывает разные варианты входа.

---

## 6. Conditional types и `infer`

`infer` позволяет извлечь неизвестную часть совпавшей структуры.

```ts
type ArrayElement<T> =
  T extends readonly (infer Element)[]
    ? Element
    : never;
```

```ts
type A = ArrayElement<string[]>;
// string

type B = ArrayElement<[number, boolean]>;
// number | boolean
```

Извлечение результата функции:

```ts
type FunctionResult<T> =
  T extends (...args: any[]) => infer Result
    ? Result
    : never;
```

```ts
function getUser() {
  return { id: 1, name: 'Alex' };
}

type User = FunctionResult<typeof getUser>;
// { id: number; name: string }
```

Подробнее `infer` разобран в отдельной шпаргалке, но для интервью важно понимать: `infer` используется именно внутри условного сопоставления типов.

---

## 7. Distributive conditional types

Если слева от `extends` находится **голый generic-параметр**, conditional type автоматически распределяется по членам union.

```ts
type ToArray<T> =
  T extends unknown
    ? T[]
    : never;
```

```ts
type Result = ToArray<string | number>;
// string[] | number[]
```

Вычисление происходит примерно так:

```ts
ToArray<string> | ToArray<number>
```

Результат:

```ts
string[] | number[]
```

Это называется **distributivity**.

### Что значит «голый параметр»

Распределение происходит здесь:

```ts
type Check<T> = T extends string
  ? A
  : B;
```

`T` непосредственно стоит слева от `extends`.

Если `T` обёрнут в другую структуру, автоматического распределения нет:

```ts
type Check<T> = [T] extends [string]
  ? A
  : B;
```

---

## 8. Как отключить distributivity

Нужно обернуть обе стороны проверки в tuple:

```ts
type ToArrayNonDistributive<T> =
  [T] extends [unknown]
    ? T[]
    : never;
```

```ts
type Result = ToArrayNonDistributive<string | number>;
// (string | number)[]
```

Сравнение:

```ts
type Distributed =
  string[] | number[];

type NonDistributed =
  (string | number)[];
```

Это разные типы:

- `string[] | number[]` — массив либо только строк, либо только чисел;
- `(string | number)[]` — один массив может содержать строки и числа одновременно.

---

## 9. Фильтрация union через `never`

Распределение и `never` позволяют фильтровать union.

```ts
type OnlyStrings<T> =
  T extends string
    ? T
    : never;
```

```ts
type Result = OnlyStrings<
  string | number | boolean
>;
// string
```

Пошагово:

```ts
OnlyStrings<string> |
OnlyStrings<number> |
OnlyStrings<boolean>
```

Получаем:

```ts
string | never | never
```

`never` исчезает из union:

```ts
string | never // string
```

Поэтому false-ветка `never` удаляет неподходящие варианты.

---

## 10. Реализация `Exclude<T, U>`

Стандартный utility type `Exclude` удаляет из `T` варианты, совместимые с `U`.

Упрощённая реализация:

```ts
type MyExclude<T, U> =
  T extends U
    ? never
    : T;
```

```ts
type Status =
  | 'idle'
  | 'loading'
  | 'success'
  | 'error';

type Finished = MyExclude<
  Status,
  'idle' | 'loading'
>;
// 'success' | 'error'
```

`T` распределяется по union, а совпавшие варианты заменяются на `never`.

---

## 11. Реализация `Extract<T, U>`

`Extract` оставляет только варианты `T`, совместимые с `U`.

```ts
type MyExtract<T, U> =
  T extends U
    ? T
    : never;
```

```ts
type Value =
  | string
  | number
  | (() => void);

type Functions = MyExtract<Value, Function>;
// () => void
```

`Exclude` и `Extract` отличаются только расположением `T` и `never` в ветках.

---

## 12. Реализация `NonNullable<T>`

```ts
type MyNonNullable<T> =
  T extends null | undefined
    ? never
    : T;
```

```ts
type Value =
  string | number | null | undefined;

type DefinedValue = MyNonNullable<Value>;
// string | number
```

Каждый член union проверяется отдельно. `null` и `undefined` заменяются на `never`.

---

## 13. Извлечение возвращаемого типа

```ts
type MyReturnType<T> =
  T extends (...args: any[]) => infer Result
    ? Result
    : never;
```

```ts
function createOrder() {
  return {
    id: 'order-1',
    status: 'pending' as const,
  };
}

type Order = MyReturnType<typeof createOrder>;
// { id: string; status: 'pending' }
```

Стандартный `ReturnType<T>` имеет constraint на функцию. При реализации собственной версии можно также ограничить `T`:

```ts
type StrictReturnType<
  T extends (...args: any[]) => any,
> = T extends (...args: any[]) => infer Result
  ? Result
  : never;
```

---

## 14. Извлечение параметров функции

```ts
type MyParameters<T> =
  T extends (...args: infer Params) => any
    ? Params
    : never;
```

```ts
function updateUser(
  id: string,
  name: string,
  active: boolean,
) {
  // ...
}

type Params = MyParameters<typeof updateUser>;
// [id: string, name: string, active: boolean]
```

Conditional type сопоставляет сигнатуру функции и извлекает tuple её параметров.

---

## 15. Извлечение значения Promise

```ts
type UnwrapPromise<T> =
  T extends PromiseLike<infer Value>
    ? Value
    : T;
```

```ts
type A = UnwrapPromise<Promise<string>>;
// string

type B = UnwrapPromise<number>;
// number
```

Рекурсивная версия:

```ts
type DeepAwaited<T> =
  T extends PromiseLike<infer Value>
    ? DeepAwaited<Value>
    : T;
```

```ts
type Result = DeepAwaited<
  Promise<Promise<User>>
>;
// User
```

В реальном коде следует использовать стандартный `Awaited<T>`, учитывающий дополнительные случаи.

---

## 16. Conditional type для объектов

Проверка наличия структуры:

```ts
type HasId<T> =
  T extends { id: unknown }
    ? true
    : false;
```

```ts
type A = HasId<{ id: number; name: string }>;
// true

type B = HasId<{ name: string }>;
// false
```

Извлечение свойства:

```ts
type IdOf<T> =
  T extends { id: infer Id }
    ? Id
    : never;
```

```ts
type A = IdOf<{ id: number }>;
// number

type B = IdOf<{ id: string }>;
// string
```

---

## 17. Выбор типа ответа по аргументу

Conditional types часто применяются в generic-функциях.

```ts
type User = {
  id: string;
  name: string;
};

type UserResult<T extends boolean> =
  T extends true
    ? User[]
    : User;
```

```ts
declare function getUsers<T extends boolean>(
  multiple: T,
): UserResult<T>;

const one = getUsers(false);
// User

const many = getUsers(true);
// User[]
```

Если аргумент имеет широкий тип `boolean`, результат тоже становится union:

```ts
declare const flag: boolean;

const result = getUsers(flag);
// User | User[]
```

Для небольшого числа вариантов перегрузки функций иногда читаются проще. Conditional return type полезен для generic API, но реализация функции может потребовать осторожной типизации.

---

## 18. Conditional types и function overloads

Вместо нескольких перегрузок:

```ts
function createLabel(value: number): IdLabel;
function createLabel(value: string): NameLabel;
function createLabel(
  value: number | string,
): IdLabel | NameLabel;
```

Можно описать зависимость через conditional type:

```ts
type Label<T> =
  T extends number
    ? IdLabel
    : NameLabel;

declare function createLabel<
  T extends number | string,
>(value: T): Label<T>;
```

```ts
const byId = createLabel(10);
// IdLabel

const byName = createLabel('Alex');
// NameLabel
```

Conditional type уменьшает количество перегрузок, если зависимость типов выражается общей формулой.

---

## 19. Nested conditional types

Можно создавать цепочки условий:

```ts
type TypeName<T> =
  T extends string
    ? 'string'
    : T extends number
      ? 'number'
      : T extends boolean
        ? 'boolean'
        : T extends (...args: any[]) => any
          ? 'function'
          : 'object';
```

```ts
type A = TypeName<string>; // 'string'
type B = TypeName<number>; // 'number'
type C = TypeName<() => void>; // 'function'
```

Слишком глубокие вложенные условия ухудшают читаемость. Для сложных решений можно:

- разбить тип на несколько вспомогательных;
- использовать lookup-структуру;
- упростить модель данных;
- добавить поясняющие имена.

---

## 20. Рекурсивные conditional types

Conditional type может ссылаться на себя.

### Глубокое извлечение элемента массива

```ts
type DeepElement<T> =
  T extends readonly (infer Element)[]
    ? DeepElement<Element>
    : T;
```

```ts
type Result = DeepElement<string[][][]>;
// string
```

### Глубокое снятие Promise

```ts
type DeepPromiseValue<T> =
  T extends PromiseLike<infer Value>
    ? DeepPromiseValue<Value>
    : T;
```

### Глубокий readonly

```ts
type DeepReadonly<T> =
  T extends (...args: any[]) => any
    ? T
    : T extends readonly unknown[]
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T extends object
        ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
        : T;
```

```ts
type ReadonlyUser = DeepReadonly<{
  profile: {
    name: string;
  };
  roles: string[];
}>;
```

Рекурсивные типы могут достигать лимита глубины вычислений и замедлять IDE, поэтому их следует применять умеренно.

---

## 21. Conditional types с mapped types

Условие можно применять к каждому свойству объекта.

### Выбрать ключи строковых свойств

```ts
type StringKeys<T> = {
  [K in keyof T]: T[K] extends string
    ? K
    : never;
}[keyof T];
```

```ts
type User = {
  id: number;
  name: string;
  email: string;
  active: boolean;
};

type UserStringKeys = StringKeys<User>;
// 'name' | 'email'
```

Пошагово:

1. Mapped type проходит по всем ключам.
2. Для строкового свойства возвращает ключ.
3. Для остальных возвращает `never`.
4. Indexed access `[keyof T]` собирает union значений.
5. `never` исчезает из union.

### Оставить только строковые свойства

```ts
type PickStringProperties<T> = {
  [K in keyof T as
    T[K] extends string ? K : never
  ]: T[K];
};

type UserStrings = PickStringProperties<User>;
// { name: string; email: string }
```

Здесь conditional type используется в key remapping.

---

## 22. Conditional types и template literal types

Можно проверять и разбирать строковые литеральные типы.

```ts
type IsEventHandler<T> =
  T extends `on${string}`
    ? true
    : false;
```

```ts
type A = IsEventHandler<'onClick'>;
// true

type B = IsEventHandler<'click'>;
// false
```

Извлечение части строки:

```ts
type EventName<T> =
  T extends `on${infer Name}`
    ? Uncapitalize<Name>
    : never;
```

```ts
type Name = EventName<'onSubmit'>;
// 'submit'
```

---

## 23. Проверка точного типа — осторожно

Условие `T extends U` проверяет одностороннюю совместимость, а не точное равенство.

```ts
type A = { id: number; name: string };
type B = { id: number };

type Result = A extends B ? true : false;
// true
```

`A` совместим с `B`, потому что содержит обязательное поле `id`.

Наивная двусторонняя проверка:

```ts
type IsEqualSimple<A, B> =
  A extends B
    ? B extends A
      ? true
      : false
    : false;
```

Она может вести себя неожиданно с union из-за distributivity.

Нераспределяющийся вариант:

```ts
type IsMutuallyAssignable<A, B> =
  [A] extends [B]
    ? [B] extends [A]
      ? true
      : false
    : false;
```

Даже взаимная assignability не во всех сложных случаях означает идентичность внутреннего представления типов. Для обычного интервью достаточно объяснить различие между совместимостью и равенством.

---

## 24. Поведение с `never`

`never` — пустой union. Поэтому distributive conditional type, получивший `never`, не имеет членов, к которым можно применить условие.

```ts
type IsNever<T> = T extends never
  ? true
  : false;

type Result = IsNever<never>;
// never, а не true
```

Для проверки `never` нужно отключить распределение:

```ts
type IsNever<T> = [T] extends [never]
  ? true
  : false;

type Result = IsNever<never>;
// true
```

Это популярный вопрос с подвохом.

---

## 25. Поведение с `any`

`any` нарушает обычную строгость системы типов и может приводить к объединению обеих веток.

```ts
type Check<T> = T extends string
  ? 'yes'
  : 'no';

type Result = Check<any>;
// 'yes' | 'no'
```

`any` означает, что компилятор не может безопасно выбрать одну ветку.

Проверка `any` требует специального паттерна:

```ts
type IsAny<T> = 0 extends (1 & T)
  ? true
  : false;
```

Это продвинутая техника. В прикладном коде лучше не строить архитектуру вокруг сложного определения `any`, а минимизировать его использование.

---

## 26. Поведение с `unknown`

`unknown` является безопасным верхним типом.

```ts
type A = string extends unknown
  ? true
  : false;
// true
```

Любой обычный тип совместим с `unknown`.

Но обратная проверка зависит от `T`:

```ts
type Check<T> = unknown extends T
  ? true
  : false;
```

```ts
type A = Check<unknown>; // true
type B = Check<string>;  // false
```

`T extends unknown` часто используется для принудительного distributive-прохода по union:

```ts
type Wrap<T> = T extends unknown
  ? { value: T }
  : never;
```

---

## 27. Поведение с `boolean`

`boolean` концептуально является union `true | false` в распределяющемся conditional type.

```ts
type Choose<T> = T extends true
  ? 'yes'
  : 'no';

type A = Choose<true>;
// 'yes'

type B = Choose<false>;
// 'no'

type C = Choose<boolean>;
// 'yes' | 'no'
```

Это объясняет, почему generic-функция с аргументом типа `boolean` часто возвращает union двух вариантов.

---

## 28. Conditional type и объединение объектов

```ts
type Event =
  | { type: 'user.created'; payload: User }
  | { type: 'order.created'; payload: Order }
  | { type: 'system.ready'; payload: null };
```

Получим события с непустым payload:

```ts
type EventsWithPayload<T> =
  T extends { payload: null }
    ? never
    : T;

type DomainEvents = EventsWithPayload<Event>;
// user.created | order.created variants
```

Получим payload конкретного имени:

```ts
type PayloadFor<
  TEvent,
  TType extends PropertyKey,
> = TEvent extends {
  type: TType;
  payload: infer Payload;
}
  ? Payload
  : never;
```

```ts
type UserPayload = PayloadFor<
  Event,
  'user.created'
>;
// User
```

Распределение проверяет каждый вариант union и оставляет совпавший.

---

## 29. Union to intersection — продвинутый пример

```ts
type UnionToIntersection<U> =
  (
    U extends unknown
      ? (value: U) => void
      : never
  ) extends (value: infer I) => void
    ? I
    : never;
```

```ts
type Result = UnionToIntersection<
  { name: string } | { age: number }
>;
// { name: string } & { age: number }
```

Идея:

1. Распределить union по функциям.
2. Поместить варианты в позицию параметра функции.
3. Извлечь совместимый тип параметра.
4. Из-за контравариантной позиции получается intersection.

Это полезный пример для продвинутого интервью, но подобные типы должны быть хорошо документированы: они сложны для чтения и поддержки.

---

## 30. Когда conditional types действительно полезны

Хорошие сценарии:

- стандартные переиспользуемые utility types;
- связь типа входа и результата generic API;
- извлечение внутреннего типа контейнера;
- фильтрация union;
- преобразование discriminated union;
- mapped types с фильтрацией ключей;
- построение типов из строковых шаблонов;
- типизация библиотечных абстракций.

Пример связи входа и результата:

```ts
type ParseResult<T> =
  T extends 'json'
    ? object
    : T extends 'text'
      ? string
      : Uint8Array;

declare function parse<T extends
  'json' | 'text' | 'binary'
>(format: T): ParseResult<T>;
```

---

## 31. Когда лучше не использовать conditional type

Не стоит усложнять типы, если:

- union можно описать напрямую;
- две понятные перегрузки читаются лучше;
- результат не должен зависеть от входного типа;
- логика существует только в runtime и не может быть доказана компилятором;
- тип создаёт трудно читаемые сообщения об ошибках;
- рекурсия замедляет IDE и сборку;
- разработчикам проекта трудно поддерживать такую абстракцию.

Плохой conditional type может сделать API формально точным, но практически неудобным.

Принцип:

> Сложность типа должна окупаться улучшением безопасности или удобства использования API.

---

## 32. Частые ошибки

### Путать `extends` с наследованием класса

В conditional type это проверка assignability.

### Не учитывать distributivity

```ts
type ToArray<T> = T extends unknown
  ? T[]
  : never;
```

Для union получится union массивов, а не массив union.

### Проверять `never` распределяющимся условием

```ts
type IsNever<T> = T extends never
  ? true
  : false;
```

Для `never` результат тоже `never`. Нужно `[T] extends [never]`.

### Ожидать точного равенства от `extends`

`A extends B` означает совместимость `A` с `B`, не обязательную идентичность.

### Использовать `any` без понимания результата

`any` может активировать обе ветки и скрыть ошибки.

### Возвращать неправильный fallback

`never`, `unknown`, `T` и конкретный тип дают разную семантику. False-ветка должна соответствовать назначению utility type.

### Чрезмерная вложенность

Многоуровневые тернарные типы трудно читать. Их лучше разбить на именованные части.

### Дублировать стандартные utility types

Реализовывать их полезно для обучения, но в рабочем коде предпочтительнее `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited` и другие встроенные типы.

### Пытаться решить runtime-валидацию

Conditional type не проверяет JSON, API-ответ или пользовательский ввод. Для этого нужны runtime guards или схемы.

---

## 33. Популярные вопросы на интервью

### Что такое conditional type?

Тип, который выбирает результат в зависимости от совместимости исходного типа с условием: `T extends U ? X : Y`.

### Что означает `extends` в условном типе?

Проверку assignability: совместим ли `T` с `U`.

### Работают ли conditional types в runtime?

Нет. Они вычисляются системой типов и удаляются из JavaScript.

### Что такое distributive conditional type?

Conditional type, который применяется отдельно к каждому члену union, когда слева от `extends` находится голый generic-параметр.

### Как отключить distributivity?

Обернуть стороны условия в tuple:

```ts
type Check<T> = [T] extends [U]
  ? X
  : Y;
```

### Зачем в conditional types используется `never`?

Для удаления вариантов из union. `never` исчезает при объединении с другими типами.

### Как реализован `Exclude<T, U>`?

```ts
type Exclude<T, U> =
  T extends U ? never : T;
```

### Как реализован `Extract<T, U>`?

```ts
type Extract<T, U> =
  T extends U ? T : never;
```

### Как реализован `NonNullable<T>`?

```ts
type NonNullable<T> =
  T extends null | undefined ? never : T;
```

### Для чего нужен `infer`?

Чтобы объявить и извлечь часть типа внутри true-ветки условного сопоставления.

### Чем constraint отличается от conditional type?

Constraint запрещает неподходящий аргумент типа. Conditional type принимает аргумент и выбирает результат в зависимости от него.

### Почему `IsNever<never>` может вернуть `never`?

Потому что распределяющийся conditional type рассматривает `never` как пустой union. Нужно отключить распределение tuple-обёрткой.

### Что произойдёт с `any`?

Часто TypeScript не может выбрать одну ветку, поэтому результат включает обе.

### Можно ли использовать conditional type рекурсивно?

Да, например для глубокого снятия Promise или массивов. Нужно учитывать лимит глубины и производительность.

---

## 34. Практические задачи с решениями

### Задача 1: проверить строку

```ts
type IsString<T> =
  T extends string ? true : false;

type A = IsString<'hello'>; // true
type B = IsString<number>;  // false
```

### Задача 2: оставить только числа

```ts
type OnlyNumbers<T> =
  T extends number ? T : never;

type Result = OnlyNumbers<
  string | number | boolean | 42
>;
// number
```

Литерал `42` поглощается широким `number` в итоговом union.

### Задача 3: удалить `null` и `undefined`

```ts
type Defined<T> =
  T extends null | undefined
    ? never
    : T;

type Result = Defined<
  string | null | number | undefined
>;
// string | number
```

### Задача 4: получить элемент массива

```ts
type Element<T> =
  T extends readonly (infer Item)[]
    ? Item
    : never;

type Result = Element<Array<User>>;
// User
```

### Задача 5: распределение по union

```ts
type Wrap<T> =
  T extends unknown ? { value: T } : never;

type Result = Wrap<string | number>;
// { value: string } | { value: number }
```

### Задача 6: отключить распределение

```ts
type WrapTogether<T> =
  [T] extends [unknown]
    ? { value: T }
    : never;

type Result = WrapTogether<string | number>;
// { value: string | number }
```

### Задача 7: получить ключи функций

```ts
type FunctionKeys<T> = {
  [K in keyof T]:
    T[K] extends (...args: any[]) => any
      ? K
      : never;
}[keyof T];

type Service = {
  url: string;
  get(): Promise<void>;
  post(data: unknown): Promise<void>;
};

type Result = FunctionKeys<Service>;
// 'get' | 'post'
```

### Задача 8: получить результат async-функции

```ts
type AsyncResult<
  T extends (...args: any[]) => any,
> = Awaited<ReturnType<T>>;

async function loadUser() {
  return { id: 1, name: 'Alex' };
}

type User = AsyncResult<typeof loadUser>;
// { id: number; name: string }
```

### Задача 9: проверить `never`

```ts
type IsNever<T> =
  [T] extends [never] ? true : false;

type A = IsNever<never>;  // true
type B = IsNever<string>; // false
```

### Задача 10: выбрать результат по флагу

```ts
type ResultByFlag<T extends boolean> =
  T extends true ? User[] : User;

type A = ResultByFlag<true>;    // User[]
type B = ResultByFlag<false>;   // User
type C = ResultByFlag<boolean>; // User[] | User
```

---

## 35. Развёрнутый ответ для собеседования

> Conditional types позволяют выбирать один тип из двух на основании проверки совместимости: `T extends U ? X : Y`. Здесь `extends` означает assignability, а не наследование класса. Условные типы работают только во время компиляции и используются для построения переиспользуемых type-level преобразований.
>
> Если слева от `extends` находится голый generic-параметр, conditional type распределяется по каждому члену union. Благодаря этому можно фильтровать union: неподходящие варианты заменяются на `never`, который исчезает из объединения. Так реализуются `Exclude`, `Extract` и `NonNullable`. Чтобы отключить распределение, обе стороны условия оборачивают в tuple: `[T] extends [U]`.
>
> Внутри conditional type можно использовать `infer`, чтобы извлечь часть структуры — например, результат функции, параметры, элемент массива или значение Promise. Conditional types также комбинируются с mapped и template literal types и могут быть рекурсивными. Важно не путать их с runtime-валидацией и контролировать сложность, потому что слишком глубокие вычисления ухудшают читаемость и производительность TypeScript.

---

## 36. Мини-шпаргалка

```ts
// Базовое условие
type IfString<T> =
  T extends string ? 'yes' : 'no';

// Фильтрация union
type OnlyStrings<T> =
  T extends string ? T : never;

// Исключение
type MyExclude<T, U> =
  T extends U ? never : T;

// Извлечение
type MyExtract<T, U> =
  T extends U ? T : never;

// Без null и undefined
type Defined<T> =
  T extends null | undefined ? never : T;

// Элемент массива
type Element<T> =
  T extends readonly (infer U)[] ? U : never;

// Результат функции
type Result<T> =
  T extends (...args: any[]) => infer R ? R : never;

// Значение Promise
type PromiseValue<T> =
  T extends PromiseLike<infer U> ? U : T;

// Отключение distributivity
type NonDistributive<T, U> =
  [T] extends [U] ? true : false;

// Проверка never
type IsNever<T> =
  [T] extends [never] ? true : false;
```

---

## 37. Чек-лист перед интервью

- Объяснить синтаксис `T extends U ? X : Y`.
- Рассказать, что `extends` означает assignability.
- Отличать conditional type от generic constraint.
- Объяснить distributive conditional types.
- Показать, как отключить distributivity.
- Объяснить роль `never` при фильтрации union.
- Реализовать `Exclude`, `Extract` и `NonNullable`.
- Использовать `infer` для результата функции и Promise.
- Объяснить поведение с `never`, `any` и `unknown`.
- Комбинировать conditional и mapped types.
- Показать recursive conditional type.
- Объяснить связь входного generic-типа с результатом функции.
- Помнить, что conditional types отсутствуют в runtime.
- Аргументировать, когда перегрузки или простой union будут понятнее.
