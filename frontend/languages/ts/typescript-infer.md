# `infer` в TypeScript — шпаргалка для собеседования

## Краткий ответ

`infer` — ключевое слово TypeScript, которое позволяет **извлечь часть типа** внутри условного типа (`conditional type`) и сохранить её во временную типовую переменную.

```ts
type UnwrapPromise<T> =
  T extends Promise<infer Value>
    ? Value
    : T;

type Result = UnwrapPromise<Promise<string>>;
// string
```

Здесь TypeScript проверяет, соответствует ли `T` шаблону `Promise<...>`. Если соответствует, тип внутри `Promise` выводится в переменную `Value`, которую можно использовать в истинной ветке.

Главная формулировка:

> `infer` объявляет типовую переменную на основании структуры сопоставляемого типа. Он используется в условии `T extends Pattern<infer X>`, чтобы извлечь `X` и применить его в true-ветке conditional type.

---

## 1. Зачем нужен `infer`

Без `infer` разработчику пришлось бы явно передавать внутренние типы или обращаться к ним специальными операциями.

Допустим, есть тип:

```ts
type ApiResponse<T> = {
  data: T;
  status: number;
};
```

Нужно получить тип `data`, зная только `ApiResponse<User>`:

```ts
type ResponseData<T> =
  T extends ApiResponse<infer Data>
    ? Data
    : never;

type UserResponseData = ResponseData<ApiResponse<User>>;
// User
```

`infer` позволяет TypeScript самостоятельно определить, какой тип находится на месте параметра `Data`.

---

## 2. Базовый синтаксис

```ts
type ExtractSomething<T> =
  T extends SomePattern<infer Result>
    ? Result
    : Fallback;
```

Части выражения:

1. `T` — исходный тип.
2. `T extends ...` — проверка соответствия шаблону.
3. `infer Result` — место, тип которого нужно вывести.
4. `Result` — выведенный тип, доступный в true-ветке.
5. `Fallback` — результат, если шаблон не подошёл.

Простой пример:

```ts
type ArrayElement<T> =
  T extends Array<infer Element>
    ? Element
    : never;

type A = ArrayElement<string[]>; // string
type B = ArrayElement<number[]>; // number
type C = ArrayElement<string>;   // never
```

---

## 3. `infer` используется внутри conditional type

Обычная форма conditional type:

```ts
type IsString<T> = T extends string ? true : false;
```

С `infer` условие не только проверяет совместимость, но и извлекает часть совпавшей структуры:

```ts
type ElementType<T> =
  T extends readonly (infer Element)[]
    ? Element
    : never;
```

Нельзя объявить `infer` как обычный псевдоним вне подходящего условного шаблона:

```ts
// type Value = infer X; // ошибка
```

Типовая переменная, созданная через `infer`, имеет локальную область видимости и обычно доступна только в истинной ветке conditional type.

---

## 4. Извлечение типа элемента массива

```ts
type ArrayElement<T> =
  T extends readonly (infer Element)[]
    ? Element
    : never;
```

```ts
type A = ArrayElement<string[]>;
// string

type B = ArrayElement<readonly number[]>;
// number

type C = ArrayElement<[string, number, boolean]>;
// string | number | boolean
```

Использование `readonly` в шаблоне позволяет поддерживать как обычные, так и readonly-массивы.

### Более короткий способ без `infer`

Для известного массива элемент можно получить через indexed access:

```ts
type Element = SomeArray[number];
```

`infer` полезен, когда нужно создать универсальный условный тип и предусмотреть fallback.

---

## 5. Извлечение возвращаемого типа функции

```ts
type MyReturnType<T> =
  T extends (...args: any[]) => infer Result
    ? Result
    : never;
```

```ts
function createUser() {
  return {
    id: 1,
    name: 'Alex',
  };
}

type User = MyReturnType<typeof createUser>;
// { id: number; name: string }
```

Логика:

1. Проверяем, является ли `T` функцией.
2. Аргументы нас не интересуют — используем `any[]`.
3. Тип результата помещаем в `Result`.
4. Возвращаем `Result`.

TypeScript уже содержит встроенный utility type `ReturnType<T>`, реализующий ту же идею.

---

## 6. Извлечение параметров функции

```ts
type MyParameters<T> =
  T extends (...args: infer Params) => unknown
    ? Params
    : never;
```

```ts
function createOrder(
  userId: string,
  amount: number,
  express: boolean,
) {
  // ...
}

type CreateOrderParams = MyParameters<typeof createOrder>;
// [userId: string, amount: number, express: boolean]
```

В `Params` выводится весь tuple параметров функции.

Первый параметр:

```ts
type FirstParameter<T> =
  T extends (first: infer First, ...args: any[]) => any
    ? First
    : never;

type UserId = FirstParameter<typeof createOrder>;
// string
```

Последний параметр:

```ts
type LastParameter<T> =
  T extends (...args: [...any[], infer Last]) => any
    ? Last
    : never;

type Express = LastParameter<typeof createOrder>;
// boolean
```

---

## 7. Извлечение типа из `Promise`

```ts
type UnwrapPromise<T> =
  T extends Promise<infer Value>
    ? Value
    : T;
```

```ts
type A = UnwrapPromise<Promise<string>>;
// string

type B = UnwrapPromise<number>;
// number
```

Такой вариант снимает только один уровень:

```ts
type Nested = UnwrapPromise<Promise<Promise<string>>>;
// Promise<string>
```

Рекурсивный вариант:

```ts
type DeepAwaited<T> =
  T extends PromiseLike<infer Value>
    ? DeepAwaited<Value>
    : T;

type Result = DeepAwaited<Promise<Promise<string>>>;
// string
```

В стандартной библиотеке TypeScript уже есть utility type `Awaited<T>`, который учитывает thenable-объекты и nullable-значения более полно. В прикладном коде обычно следует использовать его:

```ts
type Result = Awaited<ReturnType<typeof fetchUser>>;
```

---

## 8. Извлечение свойств объекта

```ts
type PropertyType<T> =
  T extends { value: infer Value }
    ? Value
    : never;
```

```ts
type A = PropertyType<{ value: string }>;
// string

type B = PropertyType<{ value: User; loading: boolean }>;
// User
```

Можно извлечь несколько свойств:

```ts
type ExtractPair<T> =
  T extends {
    key: infer Key;
    value: infer Value;
  }
    ? [Key, Value]
    : never;

type Pair = ExtractPair<{
  key: string;
  value: number;
}>;
// [string, number]
```

Если имя свойства известно заранее, часто проще использовать indexed access:

```ts
type Value = SomeType['value'];
```

`infer` нужен, когда извлечение является частью условного сопоставления структуры.

---

## 9. Работа с tuple

### Первый элемент

```ts
type Head<T extends readonly unknown[]> =
  T extends readonly [infer First, ...unknown[]]
    ? First
    : never;

type A = Head<[string, number, boolean]>;
// string
```

### Остальные элементы

```ts
type Tail<T extends readonly unknown[]> =
  T extends readonly [unknown, ...infer Rest]
    ? Rest
    : [];

type B = Tail<[string, number, boolean]>;
// [number, boolean]
```

### Последний элемент

```ts
type Last<T extends readonly unknown[]> =
  T extends readonly [...unknown[], infer Value]
    ? Value
    : never;

type C = Last<[string, number, boolean]>;
// boolean
```

### Все элементы кроме последнего

```ts
type Init<T extends readonly unknown[]> =
  T extends readonly [...infer Rest, unknown]
    ? Rest
    : [];

type D = Init<[string, number, boolean]>;
// [string, number]
```

Эти примеры используют variadic tuple types — `...infer Rest`.

---

## 10. Извлечение типа экземпляра конструктора

```ts
type MyInstanceType<T> =
  T extends abstract new (...args: any[]) => infer Instance
    ? Instance
    : never;
```

```ts
class UserService {
  getUser() {
    return { id: 1 };
  }
}

type ServiceInstance = MyInstanceType<typeof UserService>;
// UserService
```

Почему используется `typeof UserService`:

- `UserService` в позиции типа — тип экземпляра;
- `typeof UserService` — тип самого конструктора класса.

В стандартной библиотеке есть `InstanceType<T>`.

---

## 11. Извлечение параметров конструктора

```ts
type MyConstructorParameters<T> =
  T extends abstract new (...args: infer Params) => any
    ? Params
    : never;
```

```ts
class User {
  constructor(
    public id: number,
    public name: string,
  ) {}
}

type UserConstructorParams =
  MyConstructorParameters<typeof User>;
// [id: number, name: string]
```

Стандартный utility type: `ConstructorParameters<T>`.

---

## 12. Извлечение типа из generic-контейнера

```ts
type Result<TData, TError> =
  | { success: true; data: TData }
  | { success: false; error: TError };
```

Извлечём оба параметра:

```ts
type ResultParts<T> =
  T extends Result<infer Data, infer Error>
    ? [Data, Error]
    : never;
```

```ts
type Parts = ResultParts<Result<User, Error>>;
// [User, Error]
```

Можно вернуть объект:

```ts
type ResultInfo<T> =
  T extends Result<infer Data, infer Failure>
    ? {
        data: Data;
        error: Failure;
      }
    : never;
```

---

## 13. `infer` в template literal types

`infer` может извлекать части строкового литерального типа.

```ts
type EventName<T> =
  T extends `on${infer Name}`
    ? Name
    : never;
```

```ts
type A = EventName<'onClick'>;
// 'Click'

type B = EventName<'onSubmit'>;
// 'Submit'

type C = EventName<'click'>;
// never
```

Разделение строки:

```ts
type SplitOnce<T extends string> =
  T extends `${infer Left}:${infer Right}`
    ? [Left, Right]
    : [T];

type Parts = SplitOnce<'user:42'>;
// ['user', '42']
```

Извлечение параметра маршрута:

```ts
type RouteParam<T extends string> =
  T extends `${string}:${infer Param}`
    ? Param
    : never;

type Param = RouteParam<'/users/:id'>;
// 'id'
```

Для нескольких параметров потребуется рекурсивный conditional type.

---

## 14. Ограничение выведенного типа: `infer X extends ...`

Выведенный тип можно сразу ограничить:

```ts
type FirstString<T> =
  T extends readonly [infer First extends string, ...unknown[]]
    ? First
    : never;
```

```ts
type A = FirstString<['hello', 10]>;
// 'hello'

type B = FirstString<[10, 'hello']>;
// never
```

Это позволяет не писать вложенный conditional type:

```ts
type FirstStringLong<T> =
  T extends readonly [infer First, ...unknown[]]
    ? First extends string
      ? First
      : never
    : never;
```

Обе версии решают одну задачу, но `infer First extends string` компактнее.

---

## 15. Несколько `infer` в одном шаблоне

```ts
type FunctionInfo<T> =
  T extends (...args: infer Params) => infer Result
    ? {
        params: Params;
        result: Result;
      }
    : never;
```

```ts
function updateUser(
  id: string,
  patch: Partial<User>,
): Promise<User> {
  // ...
  throw new Error('Not implemented');
}

type Info = FunctionInfo<typeof updateUser>;
// {
//   params: [id: string, patch: Partial<User>];
//   result: Promise<User>;
// }
```

Можно извлечь несколько независимых частей сложной структуры за одну проверку.

---

## 16. Повторное имя `infer`

Одно имя можно использовать в нескольких позициях шаблона:

```ts
type Both<T> =
  T extends [infer Value, infer Value]
    ? Value
    : never;
```

```ts
type A = Both<[string, string]>;
// string

type B = Both<[string, number]>;
// string | number
```

Важно: это не обязательно означает «оба типа должны быть строго одинаковыми». TypeScript выводит тип, совместимый со всеми позициями; результат зависит от variance и контекста inference. Для проверки точного равенства типов обычно создают отдельный utility type.

---

## 17. Distributive conditional types

Conditional type распределяется по union, если слева от `extends` находится «голый» параметр типа.

```ts
type ArrayElement<T> =
  T extends readonly (infer Element)[]
    ? Element
    : never;
```

```ts
type Result = ArrayElement<string[] | number[]>;
// string | number
```

TypeScript применяет условие отдельно:

```ts
ArrayElement<string[]> |
ArrayElement<number[]>
```

То есть получает `string | number`.

### Как отключить распределение

Обернуть обе стороны в tuple:

```ts
type NonDistributiveElement<T> =
  [T] extends [readonly (infer Element)[]]
    ? Element
    : never;
```

Теперь union проверяется как единое целое.

Distributivity относится к conditional type, а не к самому `infer`, но часто влияет на результат извлечения.

---

## 18. Рекурсивный `infer`

Рекурсивные conditional types могут извлекать глубоко вложенные значения.

### Глубокое снятие массивов

```ts
type DeepArrayElement<T> =
  T extends readonly (infer Element)[]
    ? DeepArrayElement<Element>
    : T;
```

```ts
type Result = DeepArrayElement<string[][][]>;
// string
```

### Глубокое снятие Promise

```ts
type DeepPromiseValue<T> =
  T extends PromiseLike<infer Value>
    ? DeepPromiseValue<Value>
    : T;
```

```ts
type Result = DeepPromiseValue<
  Promise<Promise<User>>
>;
// User
```

Следует помнить об ограничении глубины вычисления типов. Чрезмерно рекурсивные типы ухудшают читаемость, скорость TypeScript и качество ошибок.

---

## 19. Пример: получить результат async-функции

```ts
async function fetchUser() {
  return {
    id: 1,
    name: 'Alex',
  };
}
```

Сначала получаем тип результата функции:

```ts
type PromiseResult = ReturnType<typeof fetchUser>;
// Promise<{ id: number; name: string }>
```

Затем снимаем Promise:

```ts
type User = Awaited<ReturnType<typeof fetchUser>>;
// { id: number; name: string }
```

Внутри `ReturnType` и `Awaited` используется логика, основанная на conditional types и `infer`.

Это частый практический сценарий: получить тип данных из уже существующей функции, не дублируя интерфейс вручную.

---

## 20. Пример: извлечь props React-компонента

Упрощённая версия:

```ts
type ComponentProps<T> =
  T extends (props: infer Props) => any
    ? Props
    : never;
```

```tsx
type ButtonProps = {
  variant: 'primary' | 'secondary';
  disabled?: boolean;
};

function Button(props: ButtonProps) {
  return <button disabled={props.disabled} />;
}

type ExtractedProps = ComponentProps<typeof Button>;
// ButtonProps
```

В React уже существует `React.ComponentProps<typeof Component>`, поддерживающий больше видов компонентов и intrinsic elements.

---

## 21. Пример: извлечь payload события

```ts
type Event<TName extends string, TPayload> = {
  type: TName;
  payload: TPayload;
};

type EventPayload<T> =
  T extends Event<string, infer Payload>
    ? Payload
    : never;
```

```ts
type UserCreated = Event<
  'user.created',
  { id: string; email: string }
>;

type Payload = EventPayload<UserCreated>;
// { id: string; email: string }
```

Если передать union событий, conditional type распределится и вернёт union всех payload:

```ts
type UserDeleted = Event<
  'user.deleted',
  { id: string }
>;

type Payloads = EventPayload<UserCreated | UserDeleted>;
// { id: string; email: string } | { id: string }
```

---

## 22. `infer` и перегруженные функции

При извлечении параметров или результата перегруженной функции TypeScript обычно использует последнюю сигнатуру перегрузки — как правило, наиболее общую реализационную сигнатуру.

```ts
declare function parse(value: string): string[];
declare function parse(value: number): number[];
declare function parse(value: string | number): string[] | number[];

type Result = ReturnType<typeof parse>;
// string[] | number[]
```

`infer` не выполняет разрешение перегрузки отдельно для желаемого аргумента. Если нужно сопоставить конкретные входы и выходы на уровне типов, часто лучше использовать generic-функцию или discriminated union.

---

## 23. `infer` vs generic-параметр

Generic-параметр объявляется заранее и передаётся или выводится при использовании типа:

```ts
type Box<T> = {
  value: T;
};
```

`infer` объявляет переменную во время сопоставления уже полученного типа:

```ts
type Unbox<T> =
  T extends Box<infer Value>
    ? Value
    : never;
```

Коротко:

- generic отвечает: «с каким типом работает структура?»;
- `infer` отвечает: «какой тип уже находится внутри этой структуры?».

---

## 24. `infer` vs `typeof`

`typeof` в позиции типа получает тип существующего значения:

```ts
const user = {
  id: 1,
  name: 'Alex',
};

type User = typeof user;
```

`infer` извлекает часть другого типа через шаблон:

```ts
type NameType<T> =
  T extends { name: infer Name }
    ? Name
    : never;

type Name = NameType<User>;
// string
```

Они часто используются вместе:

```ts
type User = Awaited<ReturnType<typeof fetchUser>>;
```

---

## 25. `infer` и `NoInfer` — разные вещи

`infer` извлекает тип внутри conditional type.

`NoInfer<T>` — встроенный utility type, который блокирует вывод типа из конкретной позиции, но сохраняет сам тип.

```ts
function choose<C extends string>(
  options: C[],
  defaultValue?: NoInfer<C>,
) {
  // ...
}
```

Здесь `C` должен выводиться из `options`, а `defaultValue` только проверяется на соответствие уже выведенному `C`.

Сходство названий не означает одинаковое назначение:

- `infer` — объявить извлекаемый тип в conditional type;
- `NoInfer<T>` — не использовать позицию как источник generic inference.

---

## 26. Что возвращать при несовпадении: `never`, `T` или fallback

Выбор false-ветки зависит от смысла utility type.

### `never`: неподходящий тип исключается

```ts
type ArrayElement<T> =
  T extends readonly (infer Element)[]
    ? Element
    : never;
```

Подходит, когда utility должен работать только с определённой структурой.

### `T`: вернуть исходный тип

```ts
type UnwrapPromise<T> =
  T extends PromiseLike<infer Value>
    ? Value
    : T;
```

Подходит для «снять обёртку, если она есть».

### Явный fallback

```ts
type ValueOrUnknown<T> =
  T extends { value: infer Value }
    ? Value
    : unknown;
```

Выбор fallback влияет на композицию utility types и поведение с union.

---

## 27. Частые ошибки

### Использовать `infer` вне conditional type

```ts
// type X = infer Value; // ошибка
```

`infer` применяется в сопоставляемом типе после `extends` conditional type.

### Думать, что `infer` работает в runtime

Все вычисления выполняются только системой типов. В JavaScript `infer` не существует.

### Изобретать встроенные utility types без необходимости

В учебных целях полезно реализовать `ReturnType`, `Parameters`, `Awaited`, `InstanceType`. В рабочем коде обычно лучше использовать стандартные варианты.

### Не учитывать `readonly`

```ts
type Element<T> =
  T extends (infer Item)[] ? Item : never;
```

Такой тип не поддержит readonly tuple. Универсальнее:

```ts
type Element<T> =
  T extends readonly (infer Item)[] ? Item : never;
```

### Не учитывать distributivity

Conditional type с голым `T` распределяется по union. Иногда это желаемое поведение, иногда нужен tuple-wrapper `[T] extends [...]`.

### Чрезмерная рекурсия

Слишком сложные recursive conditional types замедляют проверку, ухудшают сообщения об ошибках и могут достичь лимита глубины вычислений.

### Использовать `any` без причины

В позициях, результат которых не используется, часто можно выбрать `unknown`. Но для сигнатур функций variance может влиять на совместимость, поэтому заменять `any` механически тоже не следует.

### Ожидать точного равенства при повторном `infer X`

TypeScript выводит совместимый кандидат в зависимости от позиций inference. Для строгой проверки равенства нужен отдельный механизм.

---

## 28. Популярные вопросы на интервью

### Что такое `infer`?

Ключевое слово, которое объявляет типовую переменную внутри conditional type и позволяет извлечь часть сопоставляемого типа.

### Где можно использовать `infer`?

В шаблоне условного типа после `extends`, например `T extends Promise<infer U> ? U : T`.

### Существует ли `infer` в runtime?

Нет. Это исключительно механизм системы типов TypeScript.

### Что делает `infer U` в `Promise<infer U>`?

Если исходный тип совместим с `Promise<...>`, TypeScript помещает внутренний тип Promise в переменную `U`.

### Чем `infer` отличается от generic?

Generic-параметр объявляется как входной параметр типа. `infer` извлекает неизвестную часть уже полученной структуры во время conditional matching.

### Как получить возвращаемый тип функции?

```ts
type Result<T> =
  T extends (...args: any[]) => infer R
    ? R
    : never;
```

В рабочем коде используется `ReturnType<T>`.

### Как получить параметры функции?

```ts
type Params<T> =
  T extends (...args: infer P) => any
    ? P
    : never;
```

Встроенный вариант — `Parameters<T>`.

### Как получить тип из Promise?

```ts
type Value<T> =
  T extends PromiseLike<infer U>
    ? U
    : T;
```

Для рекурсивной стандартной логики используется `Awaited<T>`.

### Что происходит с union?

Если слева от `extends` находится голый generic-параметр, conditional type распределяется по каждому члену union.

### Как отключить distributivity?

Обернуть обе стороны условия в tuple:

```ts
type Check<T> = [T] extends [SomeType]
  ? TrueType
  : FalseType;
```

### Можно ли использовать несколько `infer`?

Да. Например, одновременно извлечь tuple параметров и результат функции.

### Что означает `infer U extends string`?

Это извлечение типа с одновременным ограничением: совпадение успешно только если выведенный `U` совместим со `string`.

### Как `infer` работает с перегрузками?

При стандартном извлечении обычно используется последняя, наиболее общая сигнатура перегрузки.

---

## 29. Практические задачи с решениями

### Задача 1: получить элемент массива

```ts
type Element<T> =
  T extends readonly (infer Item)[]
    ? Item
    : never;

type Result = Element<Array<{ id: number }>>;
// { id: number }
```

### Задача 2: получить результат функции

```ts
type FunctionResult<T> =
  T extends (...args: any[]) => infer Result
    ? Result
    : never;

type Result = FunctionResult<() => Promise<string>>;
// Promise<string>
```

### Задача 3: получить результат async-функции без Promise

```ts
type AsyncResult<T extends (...args: any[]) => any> =
  Awaited<ReturnType<T>>;

async function loadCount() {
  return 42;
}

type Result = AsyncResult<typeof loadCount>;
// number
```

### Задача 4: получить первый элемент tuple

```ts
type First<T extends readonly unknown[]> =
  T extends readonly [infer Head, ...unknown[]]
    ? Head
    : never;

type Result = First<['a', 1, true]>;
// 'a'
```

### Задача 5: получить хвост tuple

```ts
type Rest<T extends readonly unknown[]> =
  T extends readonly [unknown, ...infer Tail]
    ? Tail
    : [];

type Result = Rest<['a', 1, true]>;
// [1, true]
```

### Задача 6: получить тип свойства `data`

```ts
type DataOf<T> =
  T extends { data: infer Data }
    ? Data
    : never;

type Result = DataOf<{
  data: User[];
  status: number;
}>;
// User[]
```

### Задача 7: выделить имя обработчика

```ts
type HandlerName<T> =
  T extends `on${infer Name}`
    ? Name
    : never;

type Result = HandlerName<'onChange'>;
// 'Change'
```

### Задача 8: извлечь данные и ошибку

```ts
type ResultParts<T> =
  T extends {
    data: infer Data;
    error: infer Failure;
  }
    ? [Data, Failure]
    : never;

type Parts = ResultParts<{
  data: User;
  error: Error | null;
}>;
// [User, Error | null]
```

---

## 30. Готовый развёрнутый ответ для собеседования

> `infer` используется внутри conditional types для извлечения части сопоставляемого типа. Например, в выражении `T extends Promise<infer U> ? U : T` TypeScript проверяет, является ли `T` Promise-подобным типом, и если да — помещает его внутренний тип в локальную переменную `U`. Затем `U` можно вернуть или использовать для построения другого типа.
>
> Через `infer` реализуется логика стандартных utility types вроде `ReturnType`, `Parameters`, `ConstructorParameters`, `InstanceType` и `Awaited`. Он может извлекать результат и параметры функции, элементы массивов и tuple, параметры generic-контейнера и части template literal types. Допускается несколько `infer` и рекурсивное применение.
>
> Важно учитывать distributive conditional types: если слева от `extends` находится голый параметр `T`, условие применяется отдельно к каждому члену union. Распределение можно отключить через `[T] extends [...]`. `infer` работает только на уровне типов и полностью исчезает из JavaScript.

---

## 31. Мини-шпаргалка

```ts
// Элемент массива
type Element<T> =
  T extends readonly (infer U)[] ? U : never;

// Результат функции
type Result<T> =
  T extends (...args: any[]) => infer R ? R : never;

// Параметры функции
type Args<T> =
  T extends (...args: infer P) => any ? P : never;

// Значение Promise
type PromiseValue<T> =
  T extends PromiseLike<infer U> ? U : T;

// Экземпляр конструктора
type Instance<T> =
  T extends abstract new (...args: any[]) => infer I
    ? I
    : never;

// Первый элемент tuple
type First<T extends readonly unknown[]> =
  T extends readonly [infer H, ...unknown[]] ? H : never;

// Остальные элементы tuple
type Tail<T extends readonly unknown[]> =
  T extends readonly [unknown, ...infer R] ? R : [];

// Часть строки
type AfterPrefix<T> =
  T extends `prefix:${infer R}` ? R : never;
```

---

## 32. Чек-лист перед интервью

- Объяснить связь `infer` и conditional types.
- Написать `UnwrapPromise<T>`.
- Самостоятельно реализовать `ReturnType` и `Parameters`.
- Извлечь элемент массива и tuple.
- Использовать `...infer Rest`.
- Извлечь тип экземпляра конструктора.
- Использовать несколько `infer` в одном условии.
- Показать `infer` в template literal type.
- Объяснить `infer U extends Constraint`.
- Рассказать о distributive conditional types.
- Отключить distributivity через tuple-wrapper.
- Объяснить поведение с перегрузками.
- Отличать `infer` от generic-параметра, `typeof` и `NoInfer`.
- Помнить, что `infer` отсутствует в runtime.
