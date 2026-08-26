# DOM, события и работа с DOM — подготовка к интервью

## Короткий ответ для собеседования

**DOM — Document Object Model** — это объектное представление HTML-документа в виде дерева узлов. Браузер разбирает HTML и создаёт объекты `Document`, `Element`, `Text` и другие, которыми JavaScript может управлять через DOM API: искать элементы, читать и менять их свойства, создавать и удалять узлы.

События позволяют реагировать на действия пользователя и браузера. Обычно обработчики добавляют через `addEventListener`. DOM-событие проходит стадии погружения, цели и всплытия. `event.target` указывает на исходный элемент, а `event.currentTarget` — на элемент, чей обработчик выполняется сейчас. Благодаря всплытию можно использовать делегирование: установить один обработчик на общего родителя и обслуживать множество дочерних элементов.

При работе с DOM важно минимизировать лишние чтения и записи геометрии, не создавать большое количество одинаковых обработчиков, очищать подписки и не вставлять непроверенные данные через `innerHTML`, поскольку это может привести к XSS.

---

# Часть I. Теория

## 1. Что такое DOM

DOM — это программный интерфейс к документу. HTML описывает исходную разметку, а DOM представляет текущее состояние документа в памяти браузера.

```html
<main id="app">
  <h1>Hello</h1>
  <button>Save</button>
</main>
```

Упрощённое DOM-дерево:

```text
Document
└── html
    └── body
        └── main#app
            ├── h1
            │   └── Text("Hello")
            └── button
                └── Text("Save")
```

DOM не равен исходному HTML:

- браузер может исправить некорректную разметку при парсинге;
- JavaScript может изменить дерево после загрузки страницы;
- пользователь может изменить значения полей формы, не меняя HTML-атрибуты;
- DOM содержит не только элементы, но и текстовые узлы, комментарии и сам документ.

### DOM и BOM

DOM описывает документ. **BOM — Browser Object Model** — условное название браузерных API вокруг документа: `window`, `location`, `history`, `navigator`, `screen`, таймеры.

```js
window.document; // DOM
window.location; // BOM
window.history;  // BOM
```

`window` — глобальный объект вкладки, а `document` — входная точка в DOM текущей страницы.

## 2. Основные типы узлов

Все DOM-узлы реализуют интерфейс `Node`, но имеют разные возможности.

| Тип | Пример | Основная роль |
|---|---|---|
| `Document` | `document` | Корень и точка входа в документ |
| `Element` | `<div>` | HTML- или SVG-элемент |
| `Text` | текст внутри `<p>` | Текстовый узел |
| `Comment` | `<!-- note -->` | Комментарий |
| `DocumentFragment` | `document.createDocumentFragment()` | Временный контейнер для набора узлов |

Часто используемые проверки:

```js
node.nodeType === Node.ELEMENT_NODE;
node instanceof Element;
element instanceof HTMLElement;
```

`Element` содержит общие возможности элементов, а `HTMLElement` — базовый интерфейс HTML-элементов. Конкретные элементы имеют специализированные интерфейсы: `HTMLInputElement`, `HTMLButtonElement`, `HTMLFormElement`.

## 3. Поиск элементов

```js
const app = document.getElementById("app");
const button = document.querySelector("[data-action='save']");
const cards = document.querySelectorAll(".card");
```

| Метод | Что возвращает |
|---|---|
| `getElementById(id)` | Один элемент или `null` |
| `querySelector(selector)` | Первый элемент по CSS-селектору или `null` |
| `querySelectorAll(selector)` | Статический `NodeList` всех совпадений |
| `getElementsByClassName(name)` | Живой `HTMLCollection` |
| `getElementsByTagName(tag)` | Живой `HTMLCollection` |

Поиск можно выполнять относительно элемента:

```js
const list = document.querySelector(".users");
const activeItems = list.querySelectorAll(".user.active");
```

Так ограничивается область поиска и снижается риск выбрать одноимённый элемент из другой части страницы.

### `NodeList` и `HTMLCollection`

Главное интервью-различие:

- `querySelectorAll()` возвращает **статический** `NodeList`: он не обновится после изменения DOM;
- `getElementsByClassName()` и `getElementsByTagName()` возвращают **живой** `HTMLCollection`: коллекция автоматически отражает изменения дерева;
- `NodeList` может содержать разные виды узлов, а `HTMLCollection` — только элементы;
- для обычного массива можно использовать `Array.from(collection)` или `[...collection]`.

```js
const staticItems = document.querySelectorAll(".item");
const liveItems = document.getElementsByClassName("item");

document.body.append(document.createElement("div"));

// staticItems останется прежним.
// liveItems обновится, если новый div получит класс item.
```

Не все `NodeList` статические: например, `childNodes` является живой коллекцией. Поэтому лучше знать поведение конкретного API.

## 4. Навигация по DOM-дереву

### Навигация только по элементам

```js
element.parentElement;
element.children;
element.firstElementChild;
element.lastElementChild;
element.previousElementSibling;
element.nextElementSibling;
```

### Навигация по всем узлам

```js
node.parentNode;
node.childNodes;
node.firstChild;
node.lastChild;
node.previousSibling;
node.nextSibling;
```

`children` игнорирует текст и комментарии, а `childNodes` включает их. Переносы строк и пробелы в HTML могут создавать текстовые узлы, поэтому `firstChild` не обязательно является элементом.

### Поиск ближайшего предка

```js
const button = event.target.closest("button[data-action]");
```

`closest()` проверяет сам элемент и затем его предков. `matches()` проверяет соответствие элемента CSS-селектору:

```js
element.matches(".card.active");
```

## 5. Создание и изменение узлов

```js
const item = document.createElement("li");
item.classList.add("todo-item");
item.textContent = "Изучить DOM";

const list = document.querySelector(".todo-list");
list.append(item);
```

Основные методы:

| Метод | Действие |
|---|---|
| `append(...nodes)` | Добавляет в конец |
| `prepend(...nodes)` | Добавляет в начало |
| `before(...nodes)` | Вставляет перед элементом |
| `after(...nodes)` | Вставляет после элемента |
| `replaceWith(...nodes)` | Заменяет элемент |
| `remove()` | Удаляет элемент |
| `cloneNode(deep)` | Клонирует узел; `deep=true` — с потомками |
| `replaceChildren(...nodes)` | Заменяет всех дочерних узлов |

Если вставить существующий узел в другое место, он **переместится**, а не скопируется:

```js
secondContainer.append(existingElement);
```

Для копии нужен `cloneNode(true)`. Обработчики, добавленные через `addEventListener`, при `cloneNode()` не копируются.

### `DocumentFragment`

```js
const fragment = document.createDocumentFragment();

for (let i = 0; i < 100; i += 1) {
  const item = document.createElement("li");
  item.textContent = `Item ${i}`;
  fragment.append(item);
}

list.append(fragment);
```

Фрагмент удобен для сборки группы узлов до вставки. После `append(fragment)` в DOM перемещаются его дочерние узлы, а сам фрагмент остаётся пустым.

Современные браузеры и так оптимизируют многие операции, поэтому `DocumentFragment` — не магическое средство ускорения. Главное преимущество — удобная пакетная сборка дерева.

## 6. `textContent`, `innerText` и `innerHTML`

| Свойство | Особенности |
|---|---|
| `textContent` | Читает или заменяет текст узла и потомков, не интерпретирует HTML |
| `innerText` | Ориентируется на визуально отображаемый текст и может учитывать стили |
| `innerHTML` | Читает или заменяет HTML-разметку внутри элемента |

```js
element.textContent = userInput; // данные будут текстом
element.innerHTML = userInput;   // потенциальная XSS-уязвимость
```

Для непроверенных данных используйте `textContent` и создание элементов через DOM API. Если по требованиям нужно вставлять пользовательский HTML, его необходимо очищать надёжным HTML-санитайзером и применять строгую политику безопасности.

`insertAdjacentHTML(position, html)` позволяет вставить HTML в конкретную позицию без полной замены содержимого элемента:

```js
list.insertAdjacentHTML("beforeend", "<li>Safe static markup</li>");
```

Позиции: `beforebegin`, `afterbegin`, `beforeend`, `afterend`.

## 7. Атрибуты и свойства

HTML-атрибут — значение в разметке, а DOM-свойство — текущее значение объекта в памяти. Они часто отражают друг друга, но не всегда остаются синхронными.

```html
<input id="name" value="Initial">
```

```js
const input = document.querySelector("#name");

input.getAttribute("value"); // "Initial"
input.value;                 // текущее значение поля

input.value = "Changed";
input.getAttribute("value"); // всё ещё "Initial"
```

Операции с атрибутами:

```js
element.getAttribute("aria-label");
element.setAttribute("aria-label", "Закрыть");
element.hasAttribute("disabled");
element.removeAttribute("hidden");
element.toggleAttribute("hidden", shouldHide);
```

Булевы свойства удобнее менять свойствами:

```js
button.disabled = true;
checkbox.checked = false;
```

### `data-*` и `dataset`

```html
<button data-user-id="42" data-action="remove">Remove</button>
```

```js
button.dataset.userId; // "42"
button.dataset.action; // "remove"
```

Значения `dataset` являются строками. `data-user-id` преобразуется в `dataset.userId`.

## 8. Классы и стили

```js
element.classList.add("active");
element.classList.remove("loading");
element.classList.toggle("expanded", isExpanded);
element.classList.contains("active");
element.classList.replace("old", "new");
```

Для динамического состояния обычно лучше переключать CSS-классы, а не задавать много inline-стилей:

```js
element.style.width = "100px";
element.style.setProperty("--progress", "75%");

const styles = getComputedStyle(element);
console.log(styles.display);
```

`element.style` отражает inline-стили элемента. `getComputedStyle()` возвращает вычисленные стили после применения CSS-правил.

## 9. Формы

Событие формы следует обрабатывать на самой форме, а не только на кнопке:

```js
const form = document.querySelector("#login-form");

form.addEventListener("submit", async (event) => {
  event.preventDefault();

  if (!form.reportValidity()) return;

  const formData = new FormData(form);
  const values = Object.fromEntries(formData);

  await login(values);
});
```

Так код работает и при клике по submit-кнопке, и при отправке формы клавишей Enter.

### `input` и `change`

- `input` срабатывает при каждом пользовательском изменении значения;
- `change` обычно срабатывает после фиксации изменения: например, после потери фокуса текстовым полем или выбора значения;
- изменение `input.value` из JavaScript само по себе не обязано создавать пользовательское событие `input`.

## 10. События и `EventTarget`

События могут возникать из-за действий пользователя, браузера или приложения. `Element`, `Document` и `Window` реализуют `EventTarget`.

```js
function handleClick(event) {
  console.log(event.type);
}

button.addEventListener("click", handleClick);
button.removeEventListener("click", handleClick);
```

`addEventListener()` предпочтительнее `onclick`, потому что позволяет:

- добавить несколько независимых обработчиков;
- выбрать фазу события;
- использовать опции `once`, `passive` и `signal`;
- применять единый API к любому `EventTarget`.

```js
button.onclick = firstHandler;
button.onclick = secondHandler; // заменяет firstHandler

button.addEventListener("click", firstHandler);
button.addEventListener("click", secondHandler); // оба будут вызваны
```

## 11. Объект события

```js
container.addEventListener("click", (event) => {
  console.log(event.type);
  console.log(event.target);
  console.log(event.currentTarget);
  console.log(event.bubbles);
  console.log(event.cancelable);
  console.log(event.defaultPrevented);
});
```

| Свойство | Значение |
|---|---|
| `type` | Тип события, например `click` |
| `target` | Исходная цель события |
| `currentTarget` | Текущий объект, чей обработчик выполняется |
| `eventPhase` | Текущая фаза распространения |
| `bubbles` | Всплывает ли событие |
| `cancelable` | Можно ли отменить действие по умолчанию |
| `defaultPrevented` | Был ли вызван `preventDefault()` |
| `isTrusted` | Создано ли событие браузером, а не `dispatchEvent()` |

`currentTarget` имеет смысл во время выполнения обработчика. Если сохранить `event` и прочитать `currentTarget` асинхронно позже, значение обычно уже будет `null`.

## 12. Распространение события

Для события в DOM-дереве выделяют три стадии:

1. **Capturing — погружение:** событие идёт от верхних предков к цели.
2. **Target — цель:** событие достигает целевого элемента.
3. **Bubbling — всплытие:** если событие всплывает, оно идёт от цели к предкам.

```html
<div class="card">
  <button class="save">Save</button>
</div>
```

```js
const card = document.querySelector(".card");
const save = document.querySelector(".save");

card.addEventListener("click", () => console.log("card capture"), {
  capture: true,
});

save.addEventListener("click", () => console.log("button"));
card.addEventListener("click", () => console.log("card bubble"));
```

При клике по кнопке порядок будет:

```text
card capture
button
card bubble
```

По умолчанию `addEventListener` подписывает на фазу всплытия.

Не все события всплывают. Например, `focus`, `blur`, `mouseenter` и `mouseleave` не всплывают обычным способом. Для делегирования фокуса можно использовать `focusin` и `focusout`, а вместо `mouseenter`/`mouseleave` иногда подходят `mouseover`/`mouseout` с дополнительной проверкой `relatedTarget`.

## 13. `target` и `currentTarget`

```html
<button id="save">
  <span>Save</span>
</button>
```

```js
const button = document.querySelector("#save");

button.addEventListener("click", (event) => {
  console.log(event.target);        // span, если кликнули по тексту
  console.log(event.currentTarget); // button
});
```

- `target` остаётся исходной целью во время всего распространения;
- `currentTarget` меняется и указывает на объект текущего обработчика;
- в обычной функции `this === event.currentTarget`;
- стрелочная функция не получает собственный `this`.

```js
button.addEventListener("click", function (event) {
  console.log(this === event.currentTarget); // true
});
```

## 14. Отмена действия и остановка распространения

### `preventDefault()`

Отменяет стандартное действие браузера, если событие допускает отмену:

```js
link.addEventListener("click", (event) => {
  event.preventDefault(); // не переходить по ссылке
});
```

`preventDefault()` не останавливает распространение события.

### `stopPropagation()`

Не позволяет событию перейти к следующим объектам в пути распространения. Другие обработчики того же события на текущем элементе всё ещё могут выполниться.

### `stopImmediatePropagation()`

Останавливает распространение и запрещает запуск следующих обработчиков этого события на текущем элементе.

Злоупотреблять остановкой распространения не стоит: она создаёт скрытую связанность и может нарушить аналитику, модальные окна или делегирование выше по дереву.

## 15. Опции `addEventListener`

```js
const controller = new AbortController();

button.addEventListener("click", handleClick, {
  capture: false,
  once: true,
  passive: false,
  signal: controller.signal,
});

controller.abort(); // удалит обработчик
```

| Опция | Назначение |
|---|---|
| `capture` | Вызывать обработчик на фазе погружения |
| `once` | Автоматически удалить после первого вызова |
| `passive` | Обещание не вызывать `preventDefault()` |
| `signal` | Удалить обработчик при `AbortController.abort()` |

`passive: true` полезен для событий прокрутки и касаний, когда обработчик не должен блокировать стандартную прокрутку. В пассивном обработчике `preventDefault()` не сработает.

Для ручного удаления нужно передать ту же функцию и то же значение `capture`:

```js
function onClick() {}

button.addEventListener("click", onClick, { capture: true });
button.removeEventListener("click", onClick, { capture: true });
```

Так удалить обработчик не получится:

```js
button.addEventListener("click", () => console.log("click"));
button.removeEventListener("click", () => console.log("click"));
```

Это два разных объекта-функции.

## 16. Делегирование событий

Делегирование использует всплытие: один обработчик на родителе обслуживает существующие и будущие дочерние элементы.

```html
<ul id="users">
  <li data-user-id="1">
    Alex <button data-action="remove">Remove</button>
  </li>
</ul>
```

```js
const users = document.querySelector("#users");

users.addEventListener("click", (event) => {
  if (!(event.target instanceof Element)) return;

  const button = event.target.closest("button[data-action='remove']");
  if (!button || !users.contains(button)) return;

  const item = button.closest("li[data-user-id]");
  if (!item) return;

  removeUser(item.dataset.userId);
});
```

Почему недостаточно `event.target.matches("button")`? Пользователь может кликнуть по иконке или `span` внутри кнопки. `closest()` поднимается до подходящего элемента.

Проверка `users.contains(button)` важна, если обработчик установлен на контейнере, а `closest()` теоретически может найти подходящего предка за логической границей компонента.

Преимущества делегирования:

- меньше обработчиков;
- динамически добавленные элементы работают автоматически;
- централизованная логика событий.

Ограничения:

- событие должно всплывать или обрабатываться через подходящий аналог;
- логика сопоставления целей усложняется;
- один тяжёлый обработчик на слишком большом контейнере тоже может быть проблемой;
- границы Shadow DOM влияют на путь и `target`.

## 17. Пользовательские события

```js
const event = new CustomEvent("cart:add", {
  detail: { productId: "42", quantity: 1 },
  bubbles: true,
  cancelable: true,
});

const accepted = product.dispatchEvent(event);

if (!accepted) {
  console.log("Добавление было отменено");
}
```

Данные передают через `detail`. Синтетическое событие, созданное приложением, имеет `isTrusted === false`. `dispatchEvent()` вызывает обработчики синхронно и возвращает `false`, если отменяемое событие было отменено.

Пользовательские DOM-события удобны на границах независимых виджетов. Внутри обычного бизнес-кода часто проще использовать прямой вызов функции или специализированный event emitter.

## 18. События и Event Loop

Когда браузер получает пользовательское событие, соответствующий callback выполняется как задача JavaScript. Код обработчика выполняется синхронно до завершения стека.

```js
button.addEventListener("click", () => {
  console.log("1");

  Promise.resolve().then(() => console.log("3"));
  setTimeout(() => console.log("4"), 0);

  console.log("2");
});
```

После клика:

```text
1
2
3
4
```

Обработчик завершается, затем выполняются микрозадачи Promise, а таймер будет обработан в одной из следующих задач. Долгий синхронный обработчик блокирует главный поток, отрисовку и реакцию интерфейса.

## 19. Готовность DOM

Разные моменты загрузки страницы:

| Событие/состояние | Значение |
|---|---|
| Скрипт с `defer` | Выполняется после парсинга HTML и до `DOMContentLoaded` |
| `DOMContentLoaded` | DOM построен; изображения могут ещё загружаться |
| `load` на `window` | Загружена страница и зависимые ресурсы |
| `document.readyState` | `loading`, `interactive` или `complete` |

```js
document.addEventListener("DOMContentLoaded", init);
```

Если скрипт подключён с `defer` или расположен в конце `body`, DOM часто уже доступен без отдельного ожидания.

```html
<script src="app.js" defer></script>
```

## 20. Наблюдение за изменениями

### `MutationObserver`

Наблюдает за изменением дерева, атрибутов и текста:

```js
const observer = new MutationObserver((records) => {
  for (const record of records) {
    console.log(record.type);
  }
});

observer.observe(container, {
  childList: true,
  subtree: true,
  attributes: true,
});

observer.disconnect();
```

Не следует использовать `MutationObserver`, если изменение полностью контролируется вашим кодом и можно вызвать нужную функцию напрямую.

Смежные API:

- `ResizeObserver` — изменение размера элемента;
- `IntersectionObserver` — пересечение элемента с viewport или контейнером;
- `PerformanceObserver` — получение записей производительности.

## 21. Производительность при работе с DOM

DOM-операции не обязательно медленные сами по себе. Проблемы возникают из-за большого количества узлов, тяжёлых селекторов, длинных обработчиков и чередования изменений стилей с синхронным чтением геометрии.

### Layout thrashing

Плохой пример:

```js
for (const item of items) {
  item.style.width = `${item.offsetWidth + 10}px`;
}
```

Запись стиля и последующее чтение `offsetWidth` могут вынуждать браузер многократно пересчитывать layout.

Лучше сгруппировать чтения, а затем записи:

```js
const widths = [...items].map((item) => item.offsetWidth);

items.forEach((item, index) => {
  item.style.width = `${widths[index] + 10}px`;
});
```

Для визуальных изменений, привязанных к кадру, используйте `requestAnimationFrame()`:

```js
requestAnimationFrame(() => {
  element.classList.add("open");
});
```

Практические рекомендации:

- изменять классы пакетно;
- не выполнять тяжёлую работу в `scroll`, `pointermove` и `input` без необходимости;
- применять throttle/debounce там, где допустима задержка;
- использовать делегирование для больших динамических списков;
- для очень больших коллекций применять виртуализацию;
- удалять ненужные обработчики, таймеры и observers;
- не оптимизировать вслепую — измерять через Performance panel и профилировщик.

## 22. Утечки памяти

Удалённый из DOM элемент может оставаться в памяти, если на него существует ссылка из работающего приложения.

Типичные причины:

- глобальные массивы и кэши с DOM-элементами;
- неочищенные таймеры;
- обработчики на долгоживущих объектах, например `window` или `document`;
- незавершённые observers;
- замыкания, удерживающие большие поддеревья.

Удобная очистка через `AbortController`:

```js
function mountModal(modal) {
  const controller = new AbortController();
  const options = { signal: controller.signal };

  window.addEventListener("keydown", onKeyDown, options);
  modal.addEventListener("click", onModalClick, options);

  return () => controller.abort();
}
```

Обработчик на самом короткоживущем элементе не обязательно создаст утечку: если элемент и все ссылки на него стали недоступны, сборщик мусора может удалить и элемент, и обработчик. Опаснее подписки, связывающие короткоживущий компонент с долгоживущим объектом.

## 23. Безопасность

Основная опасность — DOM-based XSS:

```js
result.innerHTML = location.hash.slice(1); // опасно
```

Если злоумышленник контролирует строку, он может внедрить опасную разметку. Предпочтительно:

```js
result.textContent = location.hash.slice(1);
```

Также полезны:

- строгая Content Security Policy;
- Trusted Types в поддерживаемых сценариях;
- проверенный санитайзер для разрешённого пользовательского HTML;
- отказ от inline-обработчиков вроде `onclick="..."`;
- безопасное формирование URL и проверка допустимых протоколов.

Экранирование HTML и очистка HTML — не одно и то же. Экранирование превращает разметку в текст, а санитайзер оставляет разрешённую разметку и удаляет опасные конструкции.

## 24. Доступность и события

Если элемент должен вести себя как кнопка, используйте `<button>`, а не `<div>` с обработчиком `click`. Нативная кнопка уже поддерживает фокус, клавиатуру, роли и состояния.

```html
<button type="button" aria-expanded="false">Menu</button>
```

При изменении состояния синхронизируйте ARIA:

```js
button.setAttribute("aria-expanded", String(isOpen));
```

Основные правила:

- использовать семантические элементы;
- не удалять видимый фокус без равноценной замены;
- обеспечить управление клавиатурой;
- связывать `label` и `input`;
- использовать `keydown` для клавиш управления, но не копировать поведение нативных элементов без необходимости.

## 25. Shadow DOM и события

Shadow DOM создаёт изолированное поддерево компонента:

```js
const shadowRoot = host.attachShadow({ mode: "open" });
shadowRoot.innerHTML = `<button>Save</button>`;
```

Некоторые события пересекают границу Shadow DOM, если у них `composed === true`. При пересечении границы браузер может выполнить **retargeting**: внешний обработчик увидит в `event.target` host-элемент, а не внутреннюю кнопку.

Для просмотра доступной части фактического пути используют:

```js
event.composedPath();
```

Пользовательское событие, которое должно выйти наружу:

```js
host.dispatchEvent(
  new CustomEvent("widget:save", {
    detail: { id: "42" },
    bubbles: true,
    composed: true,
  }),
);
```

## 26. TypeScript и DOM

```ts
const form = document.querySelector<HTMLFormElement>("#login-form");
const email = document.querySelector<HTMLInputElement>("#email");

if (!form || !email) {
  throw new Error("Required form elements not found");
}

form.addEventListener("submit", (event: SubmitEvent) => {
  event.preventDefault();
  console.log(email.value);
});
```

`querySelector()` может вернуть `null`, поэтому значение нужно проверить. Не стоит без необходимости писать утверждение `!`, поскольку оно убирает проверку только для компилятора.

При делегировании `event.target` имеет тип `EventTarget | null`, а `EventTarget` не гарантирует методы элемента:

```ts
container.addEventListener("click", (event) => {
  if (!(event.target instanceof Element)) return;

  const button = event.target.closest<HTMLButtonElement>("button[data-action]");
  if (!button) return;
});
```

---

# Часть II. Вопросы для интервью

## 27. Популярные вопросы и ответы

### Что такое DOM?

DOM — объектная модель документа: дерево узлов, построенное браузером на основе HTML. JavaScript работает не с исходным HTML-файлом, а с объектами текущего DOM.

### Чем DOM отличается от HTML?

HTML — текстовая разметка, а DOM — текущее объектное дерево в памяти. DOM может отличаться из-за исправлений парсера, выполнения JavaScript и состояния элементов формы.

### Чем `Node` отличается от `Element`?

`Node` — общий интерфейс всех узлов, включая документ, текст и комментарии. `Element` — узел HTML- или SVG-элемента с атрибутами, `classList`, `matches`, `closest` и другими элементными API.

### Чем `children` отличается от `childNodes`?

`children` содержит только дочерние элементы. `childNodes` содержит все дочерние узлы, включая текст и комментарии.

### Чем `querySelectorAll` отличается от `getElementsByClassName`?

`querySelectorAll` принимает CSS-селектор и возвращает статический `NodeList`. `getElementsByClassName` ищет по классу и возвращает живой `HTMLCollection`.

### Чем атрибут отличается от свойства?

Атрибут задаётся в разметке и часто задаёт начальное состояние. Свойство находится в DOM-объекте и отражает текущее состояние. Для `input` атрибут `value` может сохранить начальную строку, а свойство `value` — текущий пользовательский ввод.

### Чем `innerHTML` отличается от `textContent`?

`innerHTML` парсит строку как разметку и опасен для непроверенных данных. `textContent` вставляет строку как текст и обычно безопаснее.

### Какие стадии проходит DOM-событие?

Погружение от предков к цели, стадия цели и, для всплывающих событий, всплытие от цели к предкам.

### Чем `target` отличается от `currentTarget`?

`target` — исходный объект события. `currentTarget` — объект, на котором выполняется текущий обработчик. При всплытии `target` остаётся прежним, а `currentTarget` меняется.

### Чем `preventDefault` отличается от `stopPropagation`?

`preventDefault` отменяет стандартное действие браузера, но не останавливает событие. `stopPropagation` останавливает дальнейшее распространение, но не отменяет стандартное действие.

### Что такое делегирование событий?

Это установка обработчика на общего предка с определением фактической цели через `event.target` и `closest`. Механизм основан на всплытии и автоматически работает для динамически добавленных потомков.

### Как удалить обработчик?

Передать в `removeEventListener` тот же тип, ту же ссылку на callback и то же значение `capture`. Альтернатива — передать `AbortSignal` при подписке и вызвать `abort()`.

### Что делает `passive: true`?

Обработчик обещает не отменять стандартное действие через `preventDefault`. Это помогает браузеру не ждать обработчик перед прокруткой, но `preventDefault()` внутри такого listener не сработает.

### Что такое layout thrashing?

Это многократные принудительные пересчёты layout из-за чередования изменений DOM/CSS и чтения геометрии. Решение — группировать чтения и записи, уменьшать объём работы и измерять производительность.

### Когда использовать `DOMContentLoaded`, а когда `load`?

`DOMContentLoaded` подходит, когда нужен построенный DOM. `load` нужен, когда необходимо дождаться зависимых ресурсов страницы, например изображений.

### В чём разница между `event` и callback?

Event — объект с данными о произошедшем событии. Callback или listener — функция, которую `EventTarget` вызывает при доставке события.

### Можно ли создать событие вручную?

Да, через `Event` или `CustomEvent`, затем вызвать `dispatchEvent`. Такое событие синтетическое и имеет `isTrusted === false`.

## 28. Вопросы с подвохом

### Всплывает ли `focus`?

Обычное событие `focus` не всплывает. Для делегирования можно использовать `focusin` либо слушать `focus` на фазе погружения.

### Копирует ли `cloneNode(true)` обработчики?

Обработчики из `addEventListener()` и присвоенные DOM-свойствам вроде `onclick` не копируются как состояние JavaScript. Inline-атрибуты разметки копируются как атрибуты.

### Создаёт ли `input.value = "x"` событие `input`?

Нет, обычное программное присваивание само по себе не имитирует пользовательский ввод. Если архитектуре действительно нужно событие, его создают явно.

### `preventDefault()` всегда работает?

Нет. Событие должно быть отменяемым (`cancelable === true`), а listener не должен быть пассивным.

### В каком порядке выполняются два обработчика на одном элементе?

Внутри одной фазы обычно в порядке регистрации. `stopImmediatePropagation()` может остановить следующие обработчики.

### Удаляется ли обработчик вместе с элементом?

Если удалённый элемент больше нигде не достижим, сборщик мусора может удалить его вместе с обработчиками. Но внешняя ссылка, таймер, observer или обработчик на долгоживущем объекте может удерживать связанные данные.

---

# Часть III. Практика

## 29. Делегирование списка

### Задача

Реализовать удаление элементов списка одним обработчиком. Элементы могут добавляться после инициализации.

```html
<ul id="tasks">
  <li data-id="1">DOM <button data-action="delete">×</button></li>
</ul>
```

### Решение

```js
const tasks = document.querySelector("#tasks");

tasks.addEventListener("click", (event) => {
  if (!(event.target instanceof Element)) return;

  const deleteButton = event.target.closest("[data-action='delete']");
  if (!deleteButton || !tasks.contains(deleteButton)) return;

  deleteButton.closest("li[data-id]")?.remove();
});
```

## 30. Исправление утечки обработчиков

### Проблема

```js
function openModal() {
  window.addEventListener("keydown", (event) => {
    if (event.key === "Escape") closeModal();
  });
}
```

При каждом открытии создаётся новый обработчик, который невозможно удалить по новой анонимной функции.

### Решение

```js
let modalController;

function openModal() {
  modalController = new AbortController();

  window.addEventListener(
    "keydown",
    (event) => {
      if (event.key === "Escape") closeModal();
    },
    { signal: modalController.signal },
  );
}

function closeModal() {
  modalController?.abort();
  modalController = undefined;
}
```

## 31. Безопасный вывод данных

### Задача

Вывести имя пользователя без XSS.

### Плохой вариант

```js
profile.innerHTML = `<h2>${user.name}</h2>`;
```

### Решение

```js
const heading = document.createElement("h2");
heading.textContent = user.name;
profile.replaceChildren(heading);
```

## 32. Что выведет код?

```html
<div id="outer">
  <button id="inner"><span>Click</span></button>
</div>
```

```js
const outer = document.querySelector("#outer");
const inner = document.querySelector("#inner");

outer.addEventListener("click", () => console.log("A"), true);
inner.addEventListener("click", () => console.log("B"));
outer.addEventListener("click", () => console.log("C"));
```

**Ответ:** `A`, `B`, `C`. Первый обработчик `outer` работает на погружении, затем событие достигает `inner`, затем всплывает к обычному обработчику `outer`.

## 33. Почему делегирование не работает?

### Ошибка

```js
list.addEventListener("click", (event) => {
  if (event.target.matches("button")) {
    removeItem();
  }
});
```

Если внутри кнопки находится `svg`, целью может стать SVG-элемент.

### Исправление

```js
list.addEventListener("click", (event) => {
  if (!(event.target instanceof Element)) return;

  const button = event.target.closest("button");
  if (!button || !list.contains(button)) return;

  removeItem(button);
});
```

## 34. Оптимизация изменения размеров

### Плохой вариант

```js
items.forEach((item) => {
  item.style.height = `${item.getBoundingClientRect().height + 20}px`;
});
```

### Улучшенный вариант

```js
const heights = [...items].map(
  (item) => item.getBoundingClientRect().height,
);

requestAnimationFrame(() => {
  items.forEach((item, index) => {
    item.style.height = `${heights[index] + 20}px`;
  });
});
```

Сначала выполняются все чтения геометрии, затем — все записи стилей.

## 35. Мини-задача на форму

```js
const form = document.querySelector("#profile-form");

form.addEventListener("submit", async (event) => {
  event.preventDefault();

  const submitButton = form.querySelector("button[type='submit']");
  submitButton.disabled = true;

  try {
    const response = await fetch("/api/profile", {
      method: "POST",
      body: new FormData(form),
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
  } finally {
    submitButton.disabled = false;
  }
});
```

На интервью стоит отметить:

- обрабатывается событие `submit`, поэтому работает Enter;
- стандартная отправка отменяется;
- кнопка блокируется от повторной отправки;
- состояние восстанавливается через `finally`;
- в реальном приложении следует показать доступный статус загрузки и обработать отмену запроса.

---

## 36. Готовый развёрнутый ответ

> DOM — это объектное дерево текущего документа. Все сущности являются узлами, но элементы предоставляют дополнительный API для атрибутов, классов и поиска. Для выбора элементов я обычно использую `querySelector` и `querySelectorAll`, для построения — `createElement`, `textContent`, `append` и `replaceChildren`. При работе с внешними данными избегаю прямого `innerHTML`, чтобы не создавать DOM-based XSS.
>
> События доставляются объектам `EventTarget`. DOM-событие проходит погружение, стадию цели и, если поддерживает, всплытие. `target` — исходная цель, `currentTarget` — объект текущего listener. `preventDefault` отменяет действие браузера, `stopPropagation` останавливает дальнейшее распространение, а `stopImmediatePropagation` также блокирует следующие обработчики на текущем объекте.
>
> Для динамических списков использую делегирование: один обработчик на контейнере, затем `event.target.closest()` и проверка границ через `contains()`. Жизненный цикл подписок контролирую именованными callback-функциями или `AbortController`. В производительности группирую чтение и запись геометрии, избегаю длинной синхронной работы в частых событиях, применяю `requestAnimationFrame`, throttle, debounce и виртуализацию только после измерений.

## 37. Чек-лист перед интервью

- Объяснить разницу между HTML, DOM и BOM.
- Назвать основные типы узлов.
- Отличать `Node` от `Element`, `children` от `childNodes`.
- Отличать статический `NodeList` от живого `HTMLCollection`.
- Уметь искать, создавать, перемещать, клонировать и удалять узлы.
- Объяснить атрибуты и DOM-свойства на примере `input.value`.
- Отличать `textContent`, `innerText` и `innerHTML`.
- Назвать три стадии распространения события.
- Отличать `target` и `currentTarget`.
- Отличать `preventDefault`, `stopPropagation` и `stopImmediatePropagation`.
- Объяснить опции `capture`, `once`, `passive`, `signal`.
- Написать делегирование через `closest()`.
- Объяснить очистку обработчиков и причины утечек.
- Рассказать про layout thrashing и пакетирование DOM-операций.
- Объяснить риск XSS при вставке HTML.
- Знать особенности событий на границе Shadow DOM.

## Источники

- [WHATWG DOM Standard](https://dom.spec.whatwg.org/)
- [MDN: Document Object Model](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model)
- [MDN: DOM events](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Events)
- [MDN: EventTarget.addEventListener()](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
- [MDN: Event](https://developer.mozilla.org/en-US/docs/Web/API/Event)
- [MDN: MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)
- [MDN: DocumentFragment](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment)
- [MDN: Element.innerHTML security considerations](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML#security_considerations)
