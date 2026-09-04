# Наследование в JavaScript — подготовка к интервью

## Короткий ответ для собеседования

JavaScript использует **прототипное наследование**. У обычного объекта есть внутреннее свойство `[[Prototype]]`, которое ссылается на другой объект или равно `null`. Если свойства нет у самого объекта, JavaScript ищет его выше по цепочке прототипов.

Синтаксис `class`, `extends` и `super` делает работу с наследованием удобнее, но экземпляры всё равно связаны через прототипы. Методы экземпляров класса находятся в `Class.prototype`, а не копируются в каждый объект. Наследование удобно для отношения «является разновидностью», но глубокие иерархии обычно лучше заменять композицией.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} издаёт звук`;
  }
}

class Dog extends Animal {
  speak() {
    return `${super.speak()}: гав`;
  }
}

const dog = new Dog("Рекс");

dog.speak();              // "Рекс издаёт звук: гав"
dog instanceof Dog;      // true
dog instanceof Animal;   // true
```

---

## 1. Прототипное наследование

У большинства объектов есть внутренняя ссылка `[[Prototype]]`:

```js
const animal = {
  eats: true,
  speak() {
    return "звук";
  },
};

const dog = Object.create(animal);
dog.name = "Рекс";

dog.name;   // "Рекс" — собственное свойство
dog.eats;   // true — найдено в animal
dog.speak(); // "звук" — метод найден в animal
```

При чтении `dog.eats` поиск идёт так:

1. JavaScript проверяет собственные свойства `dog`.
2. Если свойства нет, переходит к `Object.getPrototypeOf(dog)`, то есть к `animal`.
3. Затем проверяет прототип `animal` и продолжает поиск.
4. Поиск заканчивается после нахождения свойства или при достижении `null`.

Запись обычно создаёт или изменяет **собственное** свойство объекта:

```js
const parent = { value: 1 };
const child = Object.create(parent);

child.value = 2; // затеняет parent.value

console.log(child.value); // 2
delete child.value;
console.log(child.value); // 1 — снова видно свойство прототипа
```

Это называется **затенением** (`shadowing`). Исключением при присваивании могут быть, например, унаследованные сеттеры.

## 2. `[[Prototype]]`, `prototype` и `__proto__`

Это три разных понятия.

| Понятие | Что означает |
|---|---|
| `[[Prototype]]` | Внутренняя ссылка объекта на его прототип |
| `Constructor.prototype` | Обычное свойство функции-конструктора; становится прототипом экземпляров, созданных через `new` |
| `__proto__` | Устаревший исторический аксессор к `[[Prototype]]`; в прикладном коде лучше не использовать |

```js
function User(name) {
  this.name = name;
}

User.prototype.getName = function () {
  return this.name;
};

const user = new User("Анна");

Object.getPrototypeOf(user) === User.prototype; // true
User.prototype.constructor === User;            // true
```

Не каждый объект имеет свойство `prototype`:

```js
const obj = {};
obj.prototype; // undefined

const arrow = () => {};
arrow.prototype; // undefined — стрелка не является конструктором
```

Для чтения прототипа используйте `Object.getPrototypeOf()`. Менять его через `Object.setPrototypeOf()` можно, но это часто ухудшает оптимизацию; предпочтительнее сразу создавать объект с нужным прототипом через `Object.create()` или `new`.

## 3. Собственные и унаследованные свойства

```js
const parent = { inherited: 1 };
const child = Object.create(parent);
child.own = 2;
```

| Проверка | Результат | Значение |
|---|---:|---|
| `Object.hasOwn(child, "own")` | `true` | Свойство принадлежит самому объекту |
| `Object.hasOwn(child, "inherited")` | `false` | Свойство найдено в прототипе |
| `"inherited" in child` | `true` | Свойство есть у объекта или в цепочке |
| `Object.keys(child)` | `["own"]` | Только собственные перечисляемые строковые ключи |

`for...in` перебирает перечисляемые строковые свойства, включая унаследованные. Поэтому при необходимости собственных свойств добавляют `Object.hasOwn()` или используют `Object.keys()`, `Object.values()` либо `Object.entries()`.

## 4. Что делает оператор `new`

Упрощённо выражение `new User("Анна")` выполняет четыре действия:

1. Создаёт новый объект.
2. Устанавливает его `[[Prototype]]` равным `User.prototype`.
3. Вызывает `User` с новым объектом в качестве `this`.
4. Возвращает новый объект, если конструктор явно не вернул другой объект.

Приближённая модель:

```js
function emulateNew(Constructor, ...args) {
  const instance = Object.create(Constructor.prototype);
  const result = Constructor.apply(instance, args);

  const isObject =
    result !== null &&
    (typeof result === "object" || typeof result === "function");

  return isObject ? result : instance;
}
```

Если обычный конструктор вызвать без `new`, он не создаст экземпляр. Класс без `new` вообще выбросит `TypeError`.

### Почему методы размещают в прототипе

```js
function User(name) {
  this.name = name;
}

User.prototype.sayHi = function () {
  return `Привет, ${this.name}`;
};
```

Все экземпляры используют одну функцию `sayHi`. Если создавать функцию внутри конструктора, каждый экземпляр получит отдельную копию:

```js
function User(name) {
  this.name = name;
  this.sayHi = () => `Привет, ${this.name}`;
}
```

Второй вариант оправдан, когда нужна собственная функция с лексическим `this`, но требует больше памяти и не является методом прототипа.

## 5. Наследование через функции-конструкторы

До `class` типичный шаблон выглядел так:

```js
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function () {
  return `${this.name} издаёт звук`;
};

function Dog(name, breed) {
  Animal.call(this, name); // инициализация собственных полей
  this.breed = breed;
}

Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.speak = function () {
  return `${Animal.prototype.speak.call(this)}: гав`;
};
```

Важно: запись `Dog.prototype = Animal.prototype` неверна — оба конструктора начнут использовать один объект прототипа, и изменения одного повлияют на другой.

После полной замены `Dog.prototype` свойство `constructor` нужно восстановить. Оно не управляет наследованием, но обычно ожидается, что `Dog.prototype.constructor === Dog`.

## 6. `class`, `extends` и `super`

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} издаёт звук`;
  }

  static describe() {
    return "Животное";
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }

  speak() {
    return `${super.speak()}: гав`;
  }
}
```

Что нужно знать:

- `extends` связывает и прототипы экземпляров, и сами классы.
- В конструкторе производного класса необходимо вызвать `super()` до обращения к `this`.
- `super.method()` вызывает реализацию родительского прототипа с текущим `this`.
- Методы, объявленные в теле класса, находятся в `Class.prototype`, являются неперечисляемыми и разделяются экземплярами.
- Классы нельзя вызывать без `new`.
- Тело класса работает в строгом режиме.
- Объявление класса нельзя безопасно использовать до его инициализации.

Связи прототипов:

```js
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true
Object.getPrototypeOf(Dog) === Animal;                     // true
```

Первая связь обеспечивает наследование методов экземпляров, вторая — наследование статических методов:

```js
Dog.describe(); // "Животное"
```

Фраза «классы — только синтаксический сахар» допустима как упрощение: их наследование основано на прототипах. Но у классов есть дополнительная семантика — обязательный `new`, строгий режим, неперечисляемые методы, особое поведение производного конструктора и приватные элементы.

## 7. Поля, методы и приватное состояние

```js
class Counter {
  category = "ui"; // собственное поле каждого экземпляра
  #value = 0;       // приватное собственное поле

  increment() {     // метод Counter.prototype
    this.#value += 1;
  }

  get value() {
    return this.#value;
  }
}
```

- Публичные и приватные поля создаются для каждого экземпляра.
- Обычные методы хранятся в прототипе.
- Приватное имя `#value` проверяется во время выполнения и недоступно снаружи.
- Подкласс не может напрямую обратиться к приватному полю родителя. Однако унаследованный метод родителя может работать с этим полем на экземпляре подкласса после выполнения родительского конструктора.
- В JavaScript нет встроенного модификатора `protected`; он есть в TypeScript как проверка на этапе компиляции.

Стрелочный обработчик, объявленный полем класса, является собственным свойством экземпляра:

```js
class Button {
  handleClick = () => {
    console.log(this);
  };
}
```

Он сохраняет лексический `this`, но создаётся отдельно для каждого экземпляра и затеняет одноимённый метод прототипа.

## 8. Переопределение и полиморфизм

Подкласс может переопределить родительский метод:

```js
class Shape {
  area() {
    throw new Error("Метод area должен быть реализован");
  }
}

class Rectangle extends Shape {
  constructor(width, height) {
    super();
    this.width = width;
    this.height = height;
  }

  area() {
    return this.width * this.height;
  }
}

function printArea(shape) {
  console.log(shape.area());
}
```

`printArea` работает с объектом через общий контракт `area()`. Это полиморфизм: конкретная реализация определяется фактическим объектом.

JavaScript использует динамическую типизацию, поэтому наследование не обязательно для полиморфизма. Любой объект с подходящим методом может участвовать в таком вызове — это часто называют *duck typing*.

## 9. Как работает `instanceof`

В обычном случае выражение:

```js
object instanceof Constructor
```

проверяет, встречается ли `Constructor.prototype` в цепочке прототипов `object`.

```js
const dog = new Dog("Рекс", "корги");

dog instanceof Dog;    // true
dog instanceof Animal; // true
dog instanceof Object; // true
```

Близкая явная проверка:

```js
Dog.prototype.isPrototypeOf(dog); // true
```

Ограничения `instanceof`:

- результат может измениться после замены прототипа конструктора или изменения цепочки объекта;
- объекты из другого realm — например, другого `iframe` — имеют другие встроенные конструкторы;
- поведение можно настроить через `Symbol.hasInstance`;
- проверка говорит о прототипной связи, а не о форме и корректности данных.

Поэтому входные данные API лучше проверять по структуре и значениям, а для массивов использовать `Array.isArray()`, корректно работающий между realm.

## 10. Объекты без прототипа

```js
const dictionary = Object.create(null);
dictionary.key = "value";

Object.getPrototypeOf(dictionary); // null
dictionary.toString;               // undefined
```

Такой объект удобен как простой словарь: у него нет унаследованных ключей вроде `toString`. Но у него нет и методов `Object.prototype`, поэтому безопаснее вызывать универсальные операции через `Object.*`. Для полноценной коллекции ключей произвольного типа часто лучше подходит `Map`.

## 11. Наследование или композиция

Наследование подходит, когда подкласс действительно является разновидностью базового типа и может корректно использоваться вместо него.

```js
class Admin extends User {}
```

Композиция подходит, когда объект **содержит** или использует возможности других объектов:

```js
function createUser({ logger, permissions }) {
  return {
    login() {
      logger.log("login");
    },
    can(action) {
      return permissions.includes(action);
    },
  };
}
```

| Наследование | Композиция |
|---|---|
| Отношение «является» | Отношение «имеет/использует» |
| Повторное использование через базовый тип | Повторное использование через зависимости и функции |
| Удобный общий полиморфный интерфейс | Проще заменять и тестировать части |
| Может привести к сильной связанности и хрупкой глубокой иерархии | Требует явно собирать поведение |

Практическое правило: используйте небольшую стабильную иерархию, если отношение типов естественно. Для независимых возможностей и изменяемого поведения чаще выбирайте композицию.

## 12. TypeScript: что относится к рантайму

```ts
interface Serializable {
  serialize(): string;
}

abstract class Entity {
  abstract id: string;
}

class User extends Entity implements Serializable {
  id = "1";

  serialize() {
    return JSON.stringify(this);
  }
}
```

- `extends` класса создаёт реальное наследование в JavaScript-рантайме.
- `implements` проверяет соответствие интерфейсу только при компиляции и не меняет прототип.
- `interface extends` наследует типовые требования, но интерфейсы исчезают после компиляции.
- `abstract`, обычные `private` и `protected` в основном обеспечиваются TypeScript-проверками; нативное поле JavaScript `#private` защищается в рантайме.

## 13. Частые ошибки

### Общее изменяемое состояние в прототипе

```js
function Cart() {}
Cart.prototype.items = [];

const first = new Cart();
const second = new Cart();

first.items.push("book");
console.log(second.items); // ["book"] — массив общий
```

Исправление — создавать массив в конструкторе:

```js
function Cart() {
  this.items = [];
}
```

### Неверное связывание прототипов

```js
Dog.prototype = Animal.prototype; // ошибка: общий объект
```

Правильно:

```js
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
```

### Замена прототипа после создания экземпляра

```js
function User() {}

const oldUser = new User();
User.prototype = { role: "user" };
const newUser = new User();

oldUser.role; // undefined
newUser.role; // "user"
```

Старый экземпляр сохраняет ссылку на прежний объект прототипа.

### Проверка свойства только через значение

```js
if (obj.value !== undefined) {
  // Не отличает отсутствующее свойство от существующего со значением undefined.
}
```

Если важно наличие ключа, используйте `Object.hasOwn(obj, "value")` или `"value" in obj` в зависимости от того, нужно ли учитывать цепочку прототипов.

## 14. Популярные вопросы на интервью

### Что такое прототипное наследование?

Это механизм, при котором объект делегирует поиск отсутствующих свойств другому объекту, указанному в его `[[Prototype]]`. Так формируется цепочка прототипов.

### Чем `prototype` отличается от прототипа объекта?

`prototype` — обычное свойство функции-конструктора. `[[Prototype]]` — внутренняя ссылка каждого обычного объекта. После `const x = new C()` обычно выполняется `Object.getPrototypeOf(x) === C.prototype`.

### Является ли `class` настоящим классом?

Это классовый синтаксис с собственной семантикой, но наследование экземпляров по-прежнему реализовано цепочкой прототипов. Поэтому называть `class` просто сахаром можно лишь как упрощение.

### Зачем нужен `super`?

`super()` вызывает конструктор родителя в производном конструкторе и инициализирует `this`. `super.method()` вызывает родительскую реализацию метода с текущим `this`.

### Что проверяет `instanceof`?

Обычно он проверяет наличие `Constructor.prototype` в прототипной цепочке объекта, а не структуру объекта и не его данные.

### Чем `in` отличается от `Object.hasOwn()`?

`in` ищет свойство во всём объекте и его прототипах. `Object.hasOwn()` проверяет только собственное свойство.

### Наследуются ли статические методы?

Да. При `class Dog extends Animal` сам конструктор `Dog` наследует от `Animal`, поэтому доступен `Dog.someStaticMethod()`.

### Почему композицию часто предпочитают наследованию?

Она уменьшает связанность, позволяет независимо заменять поведение и не создаёт глубокую хрупкую иерархию. Наследование остаётся полезным для устойчивого отношения «является» и общего полиморфного контракта.

## 15. Задачи с разбором

### Задача 1: что выведет код?

```js
const base = { value: 1 };
const item = Object.create(base);

item.value = 2;
delete item.value;

console.log(item.value);
```

**Ответ:** `1`. Присваивание создало собственное свойство, а `delete` удалил только его. После удаления поиск продолжился в прототипе `base`.

### Задача 2: где находится метод?

```js
class User {
  greet() {
    return "hi";
  }
}

const user = new User();
```

**Ответ:** `greet` находится в `User.prototype`, а не в самом `user`.

```js
Object.hasOwn(user, "greet");          // false
Object.hasOwn(User.prototype, "greet"); // true
user.greet === User.prototype.greet;    // true
```

### Задача 3: почему код падает?

```js
class Dog extends Animal {
  constructor(name) {
    this.name = name;
    super(name);
  }
}
```

**Ответ:** в производном конструкторе нельзя обращаться к `this` до `super()`. Сначала нужно вызвать `super(name)`, затем устанавливать поля подкласса.

### Задача 4: результат `instanceof`

```js
class A {}
class B extends A {}

const value = new B();

console.log(value instanceof B);
console.log(value instanceof A);
console.log(value instanceof Object);
```

**Ответ:** три раза `true`, потому что цепочка содержит `B.prototype`, `A.prototype` и `Object.prototype`.

## 16. Развёрнутый готовый ответ

> В JavaScript наследование прототипное. У объекта есть внутренняя ссылка `[[Prototype]]`. Если свойство не найдено у самого объекта, движок последовательно ищет его в объектах этой цепочки до `null`. Свойство `prototype` функции-конструктора — это не прототип самой функции, а объект, который оператор `new` назначает прототипом создаваемого экземпляра.
>
> `class` и `extends` дают удобный синтаксис над этим механизмом. Методы экземпляров размещаются в прототипе класса, поля создаются у каждого экземпляра. `extends` связывает как прототипы экземпляров, так и сами конструкторы, поэтому наследуются и обычные, и статические методы. В производном конструкторе `super()` должен быть вызван до использования `this`, а `super.method()` позволяет дополнить родительскую реализацию.
>
> Для проверки собственных свойств я использую `Object.hasOwn`, а оператор `in` — когда нужна вся цепочка. `instanceof` обычно проверяет наличие `Constructor.prototype` в цепочке, поэтому это не универсальная валидация типа и имеет ограничения между realm. Наследование выбираю для естественного и стабильного отношения «является», а для независимых возможностей чаще использую композицию, чтобы снизить связанность.

## 17. Чек-лист перед интервью

- Объяснить поиск свойства по цепочке прототипов.
- Не путать `Constructor.prototype` и `Object.getPrototypeOf(instance)`.
- По шагам описать работу `new`.
- Реализовать наследование и через `Object.create`, и через `class extends`.
- Объяснить `super()` и `super.method()`.
- Различать собственные и унаследованные свойства.
- Объяснить ограничения `instanceof`.
- Знать, где находятся методы, поля и статические методы класса.
- Назвать ошибку общего изменяемого состояния в прототипе.
- Аргументировать выбор между наследованием и композицией.
- Отличать runtime-наследование класса от `implements` и интерфейсов TypeScript.

## Источники

- [ECMAScript Language: Functions and Classes](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html)
- [ECMAScript: Ordinary and Exotic Object Behaviours](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html)
- [MDN: Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [MDN: Object.create()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
- [MDN: instanceof](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof)
