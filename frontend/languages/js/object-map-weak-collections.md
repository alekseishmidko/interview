# Object vs Map, WeakMap и WeakSet

## Что проверяют на собеседовании

В этой теме важно не только знать API коллекций, но и понимать:

- какие значения могут быть ключами;
- как выполняется перебор;
- удерживает ли коллекция объекты в памяти;
- когда выбирать `Object`, `Map`, `WeakMap` или `WeakSet`;
- почему weak-коллекции нельзя перебирать;
- какие ошибки приводят к утечкам памяти.

---

# Object и Map

## Общая идея

И `Object`, и `Map` позволяют связать ключ со значением:

```js
const object = {
  name: 'Alex',
};

const map = new Map();
map.set('name', 'Alex');
```

Но они создавались для разных задач:

- `Object` — универсальная объектная структура с prototype, свойствами и методами;
- `Map` — специализированная коллекция пар «ключ — значение».

## Основные различия

| Характеристика | Object | Map |
| --- | --- | --- |
| Допустимые ключи | Строки и Symbols | Значения любого типа |
| Prototype по умолчанию | Есть | Не участвует в ключах коллекции |
| Количество элементов | Нужно вычислять | `map.size` |
| Перебор | Через специальные методы | Итерируемый напрямую |
| Порядок | Зависит от категории ключа | Порядок добавления |
| Частые добавления и удаления | Возможны, но это не основная специализация | API спроектирован для этого |
| Сериализация JSON | Естественная для обычных свойств | Нужное собственное преобразование |
| Ключ-объект | Преобразуется в строку | Сохраняется по identity |

---

## Ключи Object

Ключ свойства объекта может быть только строкой или символом.

```js
const object = {};

object[1] = 'number key';
object['1'] = 'string key';

console.log(object);
// { '1': 'string key' }
```

Число `1` преобразовалось в строку `'1'`, поэтому обе записи использовали один ключ.

### Объект в качестве ключа

```js
const firstUser = { id: 1 };
const secondUser = { id: 2 };
const roles = {};

roles[firstUser] = 'admin';
roles[secondUser] = 'editor';

console.log(roles);
// { '[object Object]': 'editor' }
```

Оба объекта преобразовались в одинаковую строку. Для настоящих объектных ключей нужен `Map`.

---

## Ключи Map

Ключом `Map` может быть любое значение:

```js
const map = new Map();

const user = { id: 1 };

map.set('name', 'Alex');
map.set(10, 'number');
map.set(true, 'boolean');
map.set(user, 'admin');
map.set(Symbol('id'), 'symbol');
```

Строковый и числовой ключи различаются:

```js
map.set(1, 'number');
map.set('1', 'string');

map.get(1);   // 'number'
map.get('1'); // 'string'
```

Объектные ключи сравниваются по identity:

```js
const user = { id: 1 };

map.set(user, 'admin');

map.get(user);        // 'admin'
map.get({ id: 1 });   // undefined
```

Новый объект с таким же содержимым — другой ключ.

---

## Сравнение ключей в Map

`Map` использует алгоритм `SameValueZero`.

Практические особенности:

```js
const map = new Map();

map.set(NaN, 'not a number');
map.get(NaN); // 'not a number'

map.set(0, 'zero');
map.get(-0); // 'zero'
```

- `NaN` считается тем же ключом, что и другой `NaN`;
- `0` и `-0` считаются одним ключом;
- объекты сравниваются по identity.

---

## API Map

```js
const users = new Map();

users.set(1, { name: 'Alex' });
users.set(2, { name: 'Max' });

users.get(1);       // { name: 'Alex' }
users.has(2);       // true
users.delete(2);    // true
users.size;         // 1
users.clear();
```

`set` возвращает сам `Map`, поэтому вызовы можно объединять:

```js
const permissions = new Map()
  .set('reader', ['read'])
  .set('editor', ['read', 'write']);
```

### Проверка отсутствующего значения

`get` возвращает `undefined`, но `undefined` может быть сохранённым значением:

```js
const map = new Map();
map.set('key', undefined);

map.get('key');     // undefined
map.get('missing'); // undefined
```

Для проверки существования ключа используют:

```js
map.has('key'); // true
```

---

## Перебор Map

`Map` является iterable и сохраняет порядок добавления записей.

```js
const map = new Map([
  ['first', 1],
  ['second', 2],
]);

for (const [key, value] of map) {
  console.log(key, value);
}
```

Методы перебора:

```js
map.keys();
map.values();
map.entries();
map.forEach((value, key) => {});
```

Итератор `Map` по умолчанию возвращает пары `[key, value]`:

```js
[...map];
// [['first', 1], ['second', 2]]
```

Обратите внимание на порядок аргументов `forEach`:

```js
map.forEach((value, key) => {
  console.log(key, value);
});
```

Сначала передаётся значение, затем ключ.

---

## Перебор Object

```js
const user = {
  id: 1,
  name: 'Alex',
};

Object.keys(user);    // ['id', 'name']
Object.values(user);  // [1, 'Alex']
Object.entries(user); // [['id', 1], ['name', 'Alex']]
```

Для symbol keys:

```js
Object.getOwnPropertySymbols(user);
Reflect.ownKeys(user);
```

`for...in` перебирает enumerable string keys, включая унаследованные, поэтому часто нужна проверка:

```js
for (const key in user) {
  if (Object.hasOwn(user, key)) {
    console.log(key, user[key]);
  }
}
```

---

## Порядок свойств Object

Нельзя просто говорить, что `Object` всегда перебирается строго в порядке добавления.

Для обычных собственных ключей порядок упрощённо выглядит так:

1. integer-index keys — по возрастанию;
2. остальные string keys — в порядке создания;
3. symbol keys — в порядке создания.

```js
const object = {
  10: 'ten',
  2: 'two',
  name: 'Alex',
};

Object.keys(object);
// ['2', '10', 'name']
```

Если порядок записей является основной семантикой коллекции, `Map` выражает это намерение понятнее.

---

## Prototype и конфликт ключей

Обычный объект наследуется от `Object.prototype`:

```js
const dictionary = {};

'toString' in dictionary; // true
```

Из-за prototype нужно различать наличие собственного и унаследованного свойства:

```js
Object.hasOwn(dictionary, 'toString'); // false
```

Объект без prototype:

```js
const dictionary = Object.create(null);

dictionary.name = 'Alex';
'toString' in dictionary; // false
```

Такой объект можно использовать как простой словарь со строковыми ключами, но у него отсутствуют обычные методы объекта.

`Map` не имеет конфликтов между пользовательскими ключами и свойствами prototype коллекции.

---

## Prototype pollution

Небезопасное копирование недоверенных ключей в обычный объект может привести к изменению prototype или логики приложения.

```js
for (const [key, value] of Object.entries(input)) {
  target[key] = value;
}
```

Защита зависит от сценария:

- валидировать допустимые ключи;
- использовать schema validation;
- применять `Object.hasOwn`;
- использовать `Object.create(null)` для словаря;
- использовать `Map`, если данные действительно являются коллекцией произвольных ключей;
- не выполнять небезопасный deep merge.

Выбор `Map` сам по себе не исправляет всю бизнес-логику, но убирает prototype из пространства ключей коллекции.

---

## Размер коллекции

Для `Map`:

```js
map.size;
```

Для объекта обычно считают собственные enumerable string keys:

```js
Object.keys(object).length;
```

Это не включает symbol keys и non-enumerable properties. Полный набор собственных ключей:

```js
Reflect.ownKeys(object).length;
```

---

## Производительность

`Map` спроектирован как коллекция для частого добавления, поиска и удаления произвольных ключей. Но утверждение «`Map` всегда быстрее `Object`» неверно.

Результат зависит от:

- JavaScript-движка;
- числа элементов;
- типов ключей;
- стабильности формы объекта;
- частоты добавления и удаления;
- характера перебора.

Движки хорошо оптимизируют объекты со стабильным набором известных свойств. Для производительности нужно измерять реальную нагрузку, а для архитектуры — выбирать структуру с правильной семантикой.

---

## Сериализация

Обычный объект естественно сериализуется в JSON:

```js
JSON.stringify({ id: 1, name: 'Alex' });
// '{"id":1,"name":"Alex"}'
```

`Map` напрямую не превращается в ожидаемый JSON:

```js
JSON.stringify(new Map([['name', 'Alex']]));
// '{}'
```

Преобразование в массив пар:

```js
const json = JSON.stringify([...map]);
const restored = new Map(JSON.parse(json));
```

Если ключи являются строками и безопасно преобразуются в свойства:

```js
const object = Object.fromEntries(map);
const mapAgain = new Map(Object.entries(object));
```

При преобразовании `Map` в `Object` нестроковые ключи могут изменить смысл, поэтому нужно учитывать типы ключей.

`structuredClone` поддерживает `Map` и сохраняет его как коллекцию:

```js
const cloned = structuredClone(map);
```

---

## Когда использовать Object

`Object` подходит, когда:

- моделируется сущность с известными полями;
- нужны свойства и методы;
- данные будут сериализоваться в JSON;
- используются object literals, destructuring и spread;
- структура описывается интерфейсом TypeScript.

```js
const user = {
  id: 1,
  name: 'Alex',
  role: 'admin',
};
```

---

## Когда использовать Map

`Map` подходит, когда:

- данные являются динамической коллекцией key-value;
- ключи могут быть объектами или значениями разных типов;
- нужны частые добавления и удаления;
- важен прямой `size`;
- нужен предсказуемый порядок добавления;
- коллекцию нужно удобно перебирать.

```js
const permissionsByUser = new Map();

permissionsByUser.set(userObject, ['read', 'write']);
```

---

# WeakMap

## Что такое WeakMap

`WeakMap` — коллекция пар «ключ — значение», которая не создаёт сильную ссылку на ключ.

Практически ключами обычно являются объекты. Современная спецификация также допускает non-registered symbols, но для совместимого прикладного кода чаще используют объектные ключи.

Значением может быть любое JavaScript-значение.

```js
const metadata = new WeakMap();

const user = { id: 1 };

metadata.set(user, {
  lastAccess: Date.now(),
});

metadata.get(user);
metadata.has(user);
metadata.delete(user);
```

Если объект-ключ больше нигде недостижим, его наличие в `WeakMap` не мешает сборщику мусора освободить объект. Связанная запись после этого также может быть удалена.

---

## Сильная и слабая ссылка

`Map` удерживает объект-ключ:

```js
const map = new Map();

let user = { id: 1 };
map.set(user, 'metadata');

user = null;
```

Объект остаётся достижимым через `map`.

`WeakMap` не должен удерживать ключ только из-за записи:

```js
const weakMap = new WeakMap();

let user = { id: 1 };
weakMap.set(user, 'metadata');

user = null;
```

Если других сильных ссылок нет, объект становится кандидатом на garbage collection.

Нельзя точно узнать, когда GC удалит объект. Сборкой мусора управляет движок.

---

## Почему WeakMap нельзя перебирать

У `WeakMap` нет:

- `size`;
- `keys()`;
- `values()`;
- `entries()`;
- `forEach()`;
- `clear()`;
- поддержки `for...of`.

Если бы ключи можно было перечислить, результат зависел бы от непредсказуемого момента работы garbage collector. Это сделало бы поведение программы недетерминированным и позволило бы наблюдать GC.

Чтобы проверить запись, код должен уже иметь ключ:

```js
weakMap.has(object);
weakMap.get(object);
```

---

## Сценарии WeakMap

### Метаданные для объекта

```js
const stateByElement = new WeakMap();

function initialize(element) {
  stateByElement.set(element, {
    initializedAt: Date.now(),
  });
}
```

Когда DOM-элемент больше не используется, запись не должна удерживать его в памяти.

### Кеширование результата по объекту

```js
const cache = new WeakMap();

function calculate(user) {
  if (cache.has(user)) {
    return cache.get(user);
  }

  const result = expensiveCalculation(user);
  cache.set(user, result);

  return result;
}
```

Кеш не мешает сборке объектов-ключей.

### Приватные данные

```js
const privateData = new WeakMap();

class User {
  constructor(name) {
    privateData.set(this, { name });
  }

  getName() {
    return privateData.get(this).name;
  }
}
```

Сегодня для полей класса часто удобнее `#privateField`, но `WeakMap` остаётся полезен для внешнего связывания данных с объектом.

---

# WeakSet

## Что такое WeakSet

`WeakSet` хранит уникальные garbage-collectable значения без сильного удержания. Практически это обычно объекты; современная спецификация также допускает non-registered symbols.

```js
const processedUsers = new WeakSet();

const user = { id: 1 };

processedUsers.add(user);
processedUsers.has(user);    // true
processedUsers.delete(user); // true
```

В отличие от обычного `Set`, weak-коллекция не предназначена для хранения строк, чисел, boolean, `null` или `undefined`.

```js
const weakSet = new WeakSet();

weakSet.add(42); // TypeError
```

---

## API WeakSet

```js
weakSet.add(value);
weakSet.has(value);
weakSet.delete(value);
```

Как и `WeakMap`, `WeakSet` нельзя перебирать, и у него нет `size`.

---

## Сценарии WeakSet

### Отметить обработанные объекты

```js
const processed = new WeakSet();

function processObject(object) {
  if (processed.has(object)) {
    return;
  }

  processed.add(object);
  performProcessing(object);
}
```

### Обход графа без повторной обработки

```js
function visit(node, visited = new WeakSet()) {
  if (visited.has(node)) {
    return;
  }

  visited.add(node);

  for (const child of node.children) {
    visit(child, visited);
  }
}
```

Если после обхода объекты больше не нужны, weak-коллекция сама по себе не удерживает их.

### Проверка разрешённых экземпляров

```js
const instances = new WeakSet();

class Service {
  constructor() {
    instances.add(this);
  }

  static isService(value) {
    return instances.has(value);
  }
}
```

---

# Сравнение всех коллекций

| Характеристика | Object | Map | WeakMap | WeakSet |
| --- | --- | --- | --- | --- |
| Что хранит | Свойства | Key-value | Key-value | Уникальные значения |
| Ключи/значения | Ключи: string/symbol | Ключи любого типа | Объекты; также non-registered symbols в современной спецификации | Garbage-collectable значения |
| Сильная ссылка | Да | Да | На ключ — нет | На значение — нет |
| Перебор | Да | Да | Нет | Нет |
| Размер | Вычисляется | `size` | Нет | Нет |
| Очистить всё | Удалять свойства | `clear()` | Нет | Нет |
| Основная задача | Структура сущности | Динамическая key-value коллекция | Метаданные, привязанные к объекту | Маркировка объектов |

---

## Map против WeakMap

| Map | WeakMap |
| --- | --- |
| Ключ любого типа | Garbage-collectable ключ |
| Удерживает ключ | Не удерживает ключ сильной ссылкой |
| Можно перебирать | Нельзя перебирать |
| Есть `size` и `clear` | Нет `size` и `clear` |
| Подходит для обычных коллекций | Подходит для metadata и object cache |

Если нужно показать список всех записей, узнать размер или использовать строковые ключи — нужен `Map`, а не `WeakMap`.

---

## Set против WeakSet

| Set | WeakSet |
| --- | --- |
| Любые значения | Только garbage-collectable значения |
| Сильно удерживает элементы | Не удерживает элементы сильной ссылкой |
| Можно перебирать | Нельзя перебирать |
| Есть `size` и `clear` | Нет `size` и `clear` |
| Обычная коллекция уникальных значений | Маркировка объектов без удержания памяти |

---

## Частые ошибки

### Использовать Object с объектами-ключами

Объектный ключ преобразуется в строку. Используйте `Map`.

### Считать Map обычным JSON-объектом

`JSON.stringify(map)` не сохраняет записи. Нужно явное преобразование.

### Пытаться перебрать WeakMap

Weak-коллекции намеренно не предоставляют iteration API.

### Ожидать немедленного удаления WeakMap-записи

Garbage collector работает недетерминированно. Нельзя строить бизнес-логику на моменте удаления.

### Считать WeakMap универсальным кешем

Он подходит только тогда, когда ключ является garbage-collectable значением и кеш не нужно перебирать. Для строкового URL-ключа нужен `Map` или другой кеш.

### Считать Map всегда быстрее Object

Нужно выбирать правильную семантику и измерять конкретную нагрузку.

### Хранить DOM-элементы в обычном Map бесконечно

Если `Map` живёт долго, он будет удерживать элементы. Для metadata, жизненный цикл которой связан с DOM-узлом, рассмотрите `WeakMap`.

### Использовать weak-коллекцию вместо явного cleanup

Weak-ссылки помогают памяти, но не отменяют необходимость закрывать соединения, снимать event listeners и освобождать внешние ресурсы.

---

## Вопросы и ответы для собеседования

### Чем Map отличается от Object?

`Map` принимает ключи любого типа, напрямую итерируется, хранит порядок добавления и имеет `size`. У `Object` ключами являются строки и Symbols, есть prototype, а сам объект лучше подходит для описания сущности с известными полями и JSON-сериализации.

### Почему Map удобнее для частых изменений коллекции?

У него специализированный API `set`, `get`, `has`, `delete`, `clear`, прямой `size` и нет конфликтов с prototype keys. Но реальную производительность всё равно измеряют.

### Как Map сравнивает ключи?

Через `SameValueZero`: `NaN` совпадает с `NaN`, `0` совпадает с `-0`, а объекты сравниваются по identity.

### Почему WeakMap называется weak?

Потому что коллекция не создаёт сильную ссылку на ключ. Если ключ больше нигде не достижим, он и связанная запись могут быть собраны GC.

### Почему у WeakMap нет `size` и итераторов?

Иначе программа могла бы наблюдать недетерминированную работу garbage collector. Наличие ключей менялось бы в непредсказуемый момент.

### Что может быть значением WeakMap?

Любое JavaScript-значение. Ограничение относится к ключу.

### Когда использовать WeakSet?

Когда нужно помечать объекты как посещённые, обработанные или принадлежащие группе, не удерживая их в памяти только из-за этой отметки.

### WeakMap гарантирует отсутствие утечек памяти?

Нет. Он только не удерживает ключ сильной ссылкой. Объект может оставаться достижимым из других мест, а внешние ресурсы всё равно требуют явного освобождения.

### Можно ли узнать, что ключ WeakMap уже удалён сборщиком мусора?

Нет. Чтобы вызвать `has` или `get`, нужно уже иметь сам ключ, а перечислить все ключи нельзя.

---

## Готовый ответ для интервью

> `Object` лучше использовать для сущностей с известными полями и удобной JSON-сериализацией. Его ключи — строки и Symbols, у него есть prototype, а порядок ключей зависит от их категории. `Map` предназначен для динамической key-value коллекции: принимает ключи любого типа, сохраняет порядок добавления, итерируется и имеет `size`. `WeakMap` похож на `Map`, но не удерживает объект-ключ сильной ссылкой, поэтому подходит для metadata и кеша, жизненный цикл которых связан с объектом. `WeakSet` аналогично хранит уникальные объекты без их удержания. `WeakMap` и `WeakSet` нельзя перебирать и у них нет `size`, поскольку результат зависел бы от недетерминированной работы garbage collector.

## Очень короткий ответ

> Object — структура сущности, Map — обычная динамическая коллекция, WeakMap — metadata по объекту без удержания его в памяти, WeakSet — отметка объектов без удержания. Weak-коллекции не итерируются и не имеют `size`.

## Чек-лист подготовки

Нужно уметь:

- сравнить допустимые ключи `Object` и `Map`;
- объяснить объектные ключи по identity;
- назвать методы `Map`;
- показать перебор `Map`;
- объяснить порядок ключей `Object`;
- рассказать про prototype conflicts;
- объяснить JSON-сериализацию `Map`;
- объяснить сильную и слабую ссылку;
- назвать ограничения `WeakMap` и `WeakSet`;
- объяснить отсутствие итераторов и `size`;
- привести сценарии metadata, cache и visited objects;
- объяснить, почему GC нельзя использовать как детерминированную бизнес-логику.

## Источники

- [Keyed Collections — ECMAScript specification](https://tc39.es/ecma262/multipage/keyed-collections.html)
- [`Map` — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)
- [`WeakMap` — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
- [`WeakSet` — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet)
- [Memory management — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management)
