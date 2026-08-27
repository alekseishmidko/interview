# Контекст в JavaScript: `this`, управление и потеря контекста

## Краткий ответ для собеседования

В JavaScript под контекстом чаще всего понимают значение `this` внутри функции. Для обычной функции `this` определяется не в момент объявления, а в момент вызова: важно, каким способом функция была вызвана. При вызове `object.method()` контекстом становится `object`, при обычном вызове в strict mode — `undefined`, а при использовании `call`, `apply` или `bind` контекст задаётся явно. Стрелочная функция собственного `this` не имеет и берёт его из внешней лексической области. Контекст обычно теряется, когда метод отделяют от объекта и передают как callback; сохранить его можно через `bind`, обёртку или стрелочную функцию.

---

## 1. Что означает «контекст»

На собеседовании словом «контекст» могут называть два связанных, но разных понятия.

### Контекст выполнения — Execution Context

Контекст выполнения — внутреннее окружение, в котором JavaScript выполняет код. В нём движок хранит:

- доступные переменные и функции;
- внешнее лексическое окружение;
- значение `this`;
- информацию, необходимую для выполнения текущего кода.

Контекст выполнения создаётся:

- для глобального кода;
- при каждом вызове функции;
- при выполнении `eval` — его использование обычно не рекомендуется.

Активные контексты располагаются в **стеке вызовов — Call Stack**.

```js
function first() {
  second();
}

function second() {
  console.log('Hello');
}

first();
```

Последовательность работы стека:

1. Создаётся глобальный контекст.
2. В стек добавляется контекст `first`.
3. В стек добавляется контекст `second`.
4. После завершения `second` его контекст удаляется.
5. Затем удаляется контекст `first`.

### Контекст вызова — значение `this`

`this` — специальное значение, доступное во время выполнения функции. Для обычных функций оно зависит от **формы вызова**, а не от места объявления функции.

Главный вопрос:

> Что находится слева от точки в момент вызова функции?

```js
const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

user.showName(); // Alex
```

В вызове `user.showName()` слева от точки находится `user`, поэтому `this === user`.

---

## 2. Лексическое окружение и `this` — не одно и то же

Лексическое окружение определяет, где функция ищет переменные. Оно зависит от места, где функция **объявлена**.

`this` обычной функции определяется способом, которым функция **вызвана**.

```js
const name = 'global';

function show() {
  const message = 'hello';
  console.log(message);   // ищется лексически
  console.log(this.name); // зависит от способа вызова
}
```

Это одна из важнейших формулировок для интервью:

> Область видимости функции лексическая, а `this` обычной функции динамический.

---

## 3. Правила определения `this`

Удобно проверять правила в следующем приоритете:

1. Функция вызвана через `new`.
2. Контекст явно задан через `call`, `apply` или `bind`.
3. Функция вызвана как метод объекта.
4. Выполнен обычный вызов функции.
5. Если это стрелочная функция, она использует внешний `this`.

### 3.1. Обычный вызов функции

```js
'use strict';

function showThis() {
  console.log(this);
}

showThis(); // undefined
```

В strict mode при обычном вызове `this` равен `undefined`.

В старом нестрогом браузерном скрипте `this` при таком вызове может автоматически стать `window`. Полагаться на это поведение не следует.

ES-модули выполняются в strict mode автоматически:

```html
<script type="module" src="app.js"></script>
```

### 3.2. Вызов метода объекта

```js
const user = {
  name: 'Alex',
  greet() {
    console.log(`Hello, ${this.name}`);
  },
};

user.greet(); // Hello, Alex
```

Важно: объект не «владеет» контекстом навсегда. Контекст появляется только в момент вызова `user.greet()`.

Одна и та же функция может получить разные значения `this`:

```js
function showName() {
  console.log(this.name);
}

const firstUser = { name: 'Alex', showName };
const secondUser = { name: 'Max', showName };

firstUser.showName();  // Alex
secondUser.showName(); // Max
```

### 3.3. Вложенные объекты

Контекстом становится объект непосредственно слева от точки:

```js
const company = {
  name: 'Acme',
  employee: {
    name: 'Alex',
    showName() {
      console.log(this.name);
    },
  },
};

company.employee.showName(); // Alex
```

Здесь `this` — `company.employee`, а не `company`.

### 3.4. Вызов с `new`

```js
function User(name) {
  this.name = name;
}

const user = new User('Alex');
console.log(user.name); // Alex
```

При вызове через `new` происходит следующее:

1. Создаётся новый объект.
2. Его прототип связывается с `User.prototype`.
3. Функция вызывается с новым объектом в качестве `this`.
4. Если функция явно не вернула другой объект, возвращается созданный объект.

Правило `new` имеет более высокий приоритет, чем ранее привязанный через `bind` объект:

```js
function User(name) {
  this.name = name;
}

const context = {};
const BoundUser = User.bind(context);
const user = new BoundUser('Alex');

console.log(user.name);    // Alex
console.log(context.name); // undefined
```

---

## 4. Как контекст теряется

Контекст теряется, когда метод отделяют от объекта. В переменную передаётся сама функция, но информация об объекте слева от точки не сохраняется.

```js
const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

const show = user.showName;
show(); // TypeError или undefined — зависит от кода и режима
```

Это не означает, что `this` «забыл» объект. Связь с объектом никогда не была постоянной: она существовала только в выражении `user.showName()`.

### Деструктуризация метода

```js
const { showName } = user;
showName(); // контекст потерян
```

### Передача метода как callback

```js
setTimeout(user.showName, 1000); // контекст потерян
```

`setTimeout` получает обычную функцию и позднее вызывает её самостоятельно, а не как `user.showName()`.

### Передача метода в метод массива

```js
const formatter = {
  prefix: 'ID:',
  format(value) {
    return `${this.prefix} ${value}`;
  },
};

const result = [1, 2, 3].map(formatter.format);
// this внутри format не равен formatter
```

### Присваивание метода другому объекту

```js
const anotherUser = {
  name: 'Max',
  showName: user.showName,
};

anotherUser.showName(); // Max
```

Функция не запоминает исходный объект. Новый способ вызова создаёт новый `this`.

---

## 5. Как сохранить или задать контекст

### 5.1. `bind`

`bind` создаёт новую функцию с постоянно привязанным `this`.

```js
const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

const boundShowName = user.showName.bind(user);
boundShowName(); // Alex
```

`bind` не вызывает функцию немедленно. Он возвращает новую функцию.

Можно заранее привязать и аргументы — это называется частичным применением:

```js
function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

const alex = { name: 'Alex' };
const sayHello = greet.bind(alex, 'Hello');

sayHello('!'); // Hello, Alex!
```

Особенности `bind`:

- возвращает новый объект-функцию;
- исходную функцию не изменяет;
- повторный `bind` не заменяет уже привязанный `this`;
- для удаления обработчика события нужно сохранить ссылку на результат `bind`.

```js
const bound = user.showName.bind(user);

button.addEventListener('click', bound);
button.removeEventListener('click', bound);
```

Так работать не будет, потому что каждый `bind` создаёт новую функцию:

```js
button.addEventListener('click', user.showName.bind(user));
button.removeEventListener('click', user.showName.bind(user));
```

### 5.2. Обёртка

Часто самый читаемый вариант — вызвать метод внутри другой функции:

```js
setTimeout(() => user.showName(), 1000);
```

Когда callback выполнится, внутри него явно произойдёт вызов `user.showName()`, поэтому контекст сохранится.

Но обёртка обращается к переменной `user` в момент выполнения:

```js
let user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

setTimeout(() => user.showName(), 1000);
user = null;

// Позднее возникнет ошибка: user уже равен null.
```

`bind` в такой ситуации фиксирует конкретный объект заранее.

### 5.3. `call`

`call` немедленно вызывает функцию, передавая контекст первым аргументом, а остальные аргументы — через запятую.

```js
function introduce(role, company) {
  console.log(`${this.name}, ${role} at ${company}`);
}

const user = { name: 'Alex' };

introduce.call(user, 'developer', 'Acme');
// Alex, developer at Acme
```

### 5.4. `apply`

`apply` также немедленно вызывает функцию, но аргументы принимает массивом или массивоподобным объектом.

```js
introduce.apply(user, ['developer', 'Acme']);
```

Сравнение:

| Метод | Вызывает сразу | Как принимает аргументы | Возвращает |
| --- | --- | --- | --- |
| `call` | Да | Через запятую | Результат вызова |
| `apply` | Да | Массивом | Результат вызова |
| `bind` | Нет | Можно частично привязать | Новую функцию |

В современном коде вместо многих случаев `apply` можно использовать spread:

```js
introduce.call(user, ...['developer', 'Acme']);
```

### 5.5. Специальный аргумент `thisArg`

Некоторые встроенные методы позволяют передать контекст отдельным аргументом:

```js
const formatter = {
  prefix: 'ID:',
  format(value) {
    return `${this.prefix} ${value}`;
  },
};

const result = [1, 2, 3].map(formatter.format, formatter);

console.log(result); // ['ID: 1', 'ID: 2', 'ID: 3']
```

Такой аргумент есть, например, у `map`, `filter`, `forEach`, `find`, `some` и `every`, но нет у всех callback API.

---

## 6. Стрелочные функции и `this`

Стрелочная функция не создаёт собственный `this`. Она берёт его из внешней лексической области.

```js
const user = {
  name: 'Alex',
  showNameLater() {
    setTimeout(() => {
      console.log(this.name);
    }, 1000);
  },
};

user.showNameLater(); // Alex
```

Стрелка создана во время выполнения `showNameLater`, поэтому использует его `this`.

### Когда стрелка ломает метод объекта

```js
const user = {
  name: 'Alex',
  showName: () => {
    console.log(this.name);
  },
};

user.showName(); // не Alex
```

Объектный литерал не создаёт отдельного контекста выполнения. Стрелка берёт `this` из внешнего кода, а не из `user`.

Для обычного метода объекта предпочтительнее сокращённый синтаксис:

```js
const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};
```

### `call`, `apply` и `bind` не меняют `this` стрелки

```js
const arrow = () => console.log(this);

arrow.call({ name: 'Alex' }); // переданный объект игнорируется
```

### Стрелку нельзя вызвать через `new`

```js
const User = (name) => {
  this.name = name;
};

new User('Alex'); // TypeError: User is not a constructor
```

У стрелочной функции также нет собственного `arguments` и обычного свойства `prototype`.

---

## 7. Контекст в классах

Методы класса хранятся в прототипе и ведут себя как обычные функции. Контекст может потеряться при передаче метода отдельно.

```js
class User {
  constructor(name) {
    this.name = name;
  }

  showName() {
    console.log(this.name);
  }
}

const user = new User('Alex');
const show = user.showName;

show(); // TypeError: this равен undefined
```

Код внутри классов всегда выполняется в strict mode.

### Привязка метода в конструкторе

```js
class User {
  constructor(name) {
    this.name = name;
    this.showName = this.showName.bind(this);
  }

  showName() {
    console.log(this.name);
  }
}
```

Преимущества:

- контекст закреплён один раз;
- ссылка на функцию стабильна;
- метод можно безопасно передавать как callback.

Недостаток: для каждого экземпляра создаётся отдельная bound-функция.

### Поле класса со стрелочной функцией

```js
class User {
  constructor(name) {
    this.name = name;
  }

  showName = () => {
    console.log(this.name);
  };
}
```

Такой обработчик тоже сохраняет `this`, но создаётся отдельно для каждого экземпляра и находится на самом объекте, а не в прототипе.

### Какой вариант выбрать

- Обычный прототипный метод — по умолчанию.
- Обёртка — когда callback используется в одном месте.
- `bind` в конструкторе — когда нужна стабильная ссылка на метод.
- Поле-стрелка — удобно для callback, если допустима отдельная функция на экземпляр.

---

## 8. Контекст в обработчиках DOM-событий

В обработчике, объявленном обычной функцией, браузер обычно устанавливает `this` равным элементу, на котором зарегистрирован обработчик. Он совпадает с `event.currentTarget`.

```js
button.addEventListener('click', function (event) {
  console.log(this === event.currentTarget); // true
});
```

У стрелочной функции собственного `this` нет:

```js
button.addEventListener('click', (event) => {
  console.log(this);                // внешний this
  console.log(event.currentTarget); // button
});
```

В современном коде лучше явно использовать `event.currentTarget`: это понятнее и не зависит от вида функции.

Не путать:

- `event.target` — элемент, на котором событие возникло;
- `event.currentTarget` — элемент, обработчик которого сейчас выполняется.

---

## 9. `this` на верхнем уровне

Значение верхнеуровневого `this` зависит от среды и типа модуля.

### Обычный браузерный скрипт

```html
<script src="app.js"></script>
```

На верхнем уровне `this` обычно равно `window`.

### ES-модуль

```html
<script type="module" src="app.js"></script>
```

На верхнем уровне `this` равно `undefined`.

### Node.js CommonJS

На верхнем уровне файла `this` связано с `module.exports`, а не с глобальным объектом.

### Универсальный глобальный объект

Для обращения к глобальному объекту в разных средах используется `globalThis`:

```js
console.log(globalThis);
```

Не следует использовать верхнеуровневый `this` как универсальную ссылку на глобальный объект.

---

## 10. Частые ловушки

### Скобки сами по себе обычно не теряют контекст

```js
(user.showName)(); // Alex
```

Ссылка на свойство сохраняется. Но присваивание возвращает обычное значение-функцию:

```js
const show = user.showName;
show(); // контекст потерян
```

### Цепочка методов

```js
const calculator = {
  value: 0,
  add(number) {
    this.value += number;
    return this;
  },
  multiply(number) {
    this.value *= number;
    return this;
  },
};

calculator.add(5).multiply(2);
console.log(calculator.value); // 10
```

Цепочка работает, потому что методы возвращают объект, который затем снова оказывается слева от точки.

### Метод, возвращающий вложенную обычную функцию

```js
const user = {
  name: 'Alex',
  createPrinter() {
    return function () {
      console.log(this.name);
    };
  },
};

const print = user.createPrinter();
print(); // контекст user не сохраняется
```

Обычная вложенная функция получает `this` из собственного способа вызова. Исправление:

```js
const user = {
  name: 'Alex',
  createPrinter() {
    return () => {
      console.log(this.name);
    };
  },
};
```

### Старый приём `const self = this`

```js
function Timer() {
  const self = this;

  setTimeout(function () {
    console.log(self);
  }, 1000);
}
```

До появления стрелочных функций контекст часто сохраняли в переменной `self` или `that`. Сейчас обычно используют стрелку или `bind`.

### Опциональная цепочка

```js
user.showName?.(); // this === user
```

Такой вызов сохраняет объект как получателя метода. Но после извлечения функции контекст всё равно теряется:

```js
const show = user.showName;
show?.(); // обычный вызов
```

---

## 11. Замыкание и сохранение данных без `this`

Иногда проблему лучше решить не привязкой контекста, а замыканием.

```js
function createCounter() {
  let value = 0;

  return {
    increment() {
      value += 1;
    },
    getValue() {
      return value;
    },
  };
}

const counter = createCounter();
const increment = counter.increment;

increment();
console.log(counter.getValue()); // 1
```

Методы работают даже после отделения от объекта, потому что используют переменную из замыкания, а не `this`.

Практический вывод:

- `this` полезен для методов объектов и экземпляров классов;
- замыкание удобно для приватного состояния и функций, которые будут часто передаваться отдельно;
- явные параметры обычно проще неявного контекста.

---

## 12. Как не потерять контекст: практический алгоритм

Если метод передаётся как callback:

1. Проверьте, кто именно позднее вызывает функцию.
2. Посмотрите, останется ли объект слева от точки.
3. Если нет — выберите подходящий способ:
   - `bind`, если нужна заранее привязанная стабильная функция;
   - стрелочная обёртка, если вызов используется локально;
   - поле класса со стрелкой, если метод постоянно служит callback;
   - `thisArg`, если конкретный API его поддерживает;
   - явный параметр или замыкание, если `this` вообще не нужен.
4. Если функцию потребуется удалить из подписки, сохраните её ссылку.

```js
class Controller {
  constructor(button) {
    this.button = button;
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    console.log(this);
  }

  mount() {
    this.button.addEventListener('click', this.handleClick);
  }

  unmount() {
    this.button.removeEventListener('click', this.handleClick);
  }
}
```

---

## 13. Задачи для интервью

### Задача 1

```js
'use strict';

const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

const fn = user.showName;
fn();
```

**Ответ:** возникнет ошибка при чтении `name`, потому что при обычном вызове `fn()` в strict mode `this === undefined`.

### Задача 2

```js
const user = {
  name: 'Alex',
  showName() {
    console.log(this.name);
  },
};

const admin = {
  name: 'Max',
  showName: user.showName,
};

admin.showName();
```

**Ответ:** `Max`. Контекст определяется в момент вызова, слева от точки находится `admin`.

### Задача 3

```js
const user = {
  name: 'Alex',
  showName: () => console.log(this.name),
};

user.showName();
```

**Ответ:** не `Alex`. Стрелка не получает `this` от объекта `user`, а использует `this` внешней области.

### Задача 4

```js
function show(a, b) {
  console.log(this.value, a, b);
}

const bound = show.bind({ value: 10 }, 20);
bound(30);
```

**Ответ:** `10 20 30`. `bind` зафиксировал контекст и первый аргумент.

### Задача 5

```js
const object = {
  value: 10,
  method() {
    return () => this.value;
  },
};

const fn = object.method();
console.log(fn.call({ value: 20 }));
```

**Ответ:** `10`. Стрелка получила `this` из вызова `object.method()`, а `call` не может изменить `this` стрелочной функции.

---

## 14. Популярные вопросы и ответы

### От чего зависит `this`?

У обычной функции — от способа вызова. У стрелочной — от внешней лексической области.

### Почему при передаче метода callback-контекст теряется?

Потому что передаётся значение-функция, а позднее она вызывается без исходного объекта слева от точки.

### Чем `bind` отличается от `call` и `apply`?

`call` и `apply` вызывают функцию сразу. `bind` создаёт новую функцию, которую можно вызвать позже.

### Чем `call` отличается от `apply`?

`call` принимает аргументы через запятую, `apply` — массивом или массивоподобным объектом.

### Можно ли изменить `this` стрелочной функции через `bind`?

Нет. Стрелка лексически захватывает внешний `this`.

### Есть ли у объекта собственный контекст?

Нет. `this` появляется при вызове функции. Метод — это обычная функция, хранящаяся в свойстве объекта.

### Где лучше использовать стрелочные функции?

В коротких callback и вложенных функциях, которым нужен `this` внешнего метода. Для методов объектного литерала обычно нужна обычная функция.

### Почему `this` в классе иногда равен `undefined`?

Метод класса был вызван отдельно от экземпляра. Кроме того, код класса всегда выполняется в strict mode.

### Можно ли навсегда привязать контекст?

`bind` создаёт функцию с привязанным `this`. Обычный повторный вызов через `call` или `apply` эту привязку не заменит, хотя вызов bound-функции через `new` создаст новый объект.

---

## 15. Итоговая памятка

```js
fn();                    // undefined в strict mode
object.fn();             // this === object
fn.call(object, a, b);   // явный this, немедленный вызов
fn.apply(object, [a, b]); // явный this, немедленный вызов
fn.bind(object);         // новая функция с привязанным this
new Fn();                // this === новый объект
() => this;              // this берётся снаружи
```

Главное правило:

> Для обычной функции смотрите на место её вызова, а не на место объявления.

Чтобы не потерять контекст при передаче метода, используйте `bind`, стрелочную обёртку, стабильное поле-стрелку либо откажитесь от `this` в пользу явных параметров или замыкания.
