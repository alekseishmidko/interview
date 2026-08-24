# Асинхронность JavaScript: практические задачи

Практикум для подготовки к задачам на порядок выполнения JavaScript-кода. Сначала решите каждую задачу самостоятельно, а затем раскройте блок с ответом и сверьте объяснение.

## 11. Задачи на порядок логов

На собеседовании решайте такие задачи по алгоритму:

1. Выпишите весь синхронный код.
2. Отдельно запишите микрозадачи в порядке добавления.
3. Отдельно запишите задачи таймеров и событий.
4. Выполните весь синхронный код.
5. Полностью очистите очередь микрозадач, учитывая новые микрозадачи.
6. Возьмите следующую задачу и повторите процесс.

### Задача 1: Promise и таймер

```js
console.log(1);

setTimeout(() => console.log(2), 0);

Promise.resolve().then(() => console.log(3));

console.log(4);
```

<details>
<summary>Ответ</summary>

```text
1
4
3
2
```

Сначала синхронные логи, затем Promise-микрозадача, потом задача таймера.

</details>

### Задача 2: синхронный executor

```js
console.log('A');

new Promise((resolve) => {
  console.log('B');
  resolve();
}).then(() => console.log('C'));

console.log('D');
```

<details>
<summary>Ответ</summary>

```text
A
B
D
C
```

Executor запускается синхронно, а `.then` — как микрозадача.

</details>

### Задача 3: `async/await`

```js
async function run() {
  console.log(2);
  await 0;
  console.log(4);
}

console.log(1);
run();
console.log(3);
```

<details>
<summary>Ответ</summary>

```text
1
2
3
4
```

Код до `await` выполняется синхронно. Продолжение после `await` становится микрозадачей.

</details>

### Задача 4: микрозадачи, созданные микрозадачей

```js
console.log(1);

Promise.resolve().then(() => {
  console.log(3);
  queueMicrotask(() => console.log(5));
});

queueMicrotask(() => console.log(4));

setTimeout(() => console.log(6), 0);

console.log(2);
```

<details>
<summary>Ответ</summary>

```text
1
2
3
4
5
6
```

Сначала в очереди находятся микрозадачи с логами `3` и `4`. Во время первой создаётся микрозадача `5`, которая добавляется в конец очереди.

</details>

### Задача 5: цепочка Promise

```js
Promise.resolve()
  .then(() => {
    console.log(1);
    return 2;
  })
  .then((value) => console.log(value));

Promise.resolve().then(() => console.log(3));

console.log(4);
```

<details>
<summary>Ответ</summary>

```text
4
1
3
2
```

Первый `.then` и отдельный `.then` изначально стоят в очереди. Второй обработчик цепочки добавится только после выполнения первого, поэтому окажется после лога `3`.

</details>

### Задача 6: таймер создаёт микрозадачу

```js
setTimeout(() => {
  console.log(1);
  Promise.resolve().then(() => console.log(2));
}, 0);

setTimeout(() => console.log(3), 0);
```

<details>
<summary>Ответ</summary>

```text
1
2
3
```

После первой задачи таймера Event Loop очищает очередь микрозадач и только потом берёт второй таймер.

</details>

### Задача 7: ошибка в цепочке

```js
Promise.resolve()
  .then(() => {
    console.log(1);
    throw new Error('failed');
  })
  .then(() => console.log(2))
  .catch(() => console.log(3))
  .finally(() => console.log(4));
```

<details>
<summary>Ответ</summary>

```text
1
3
4
```

После ошибки успешный обработчик с логом `2` пропускается. Ошибку перехватывает `.catch`, после чего выполняется `.finally`.

</details>

### Задача 8: цикл и `var`

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

<details>
<summary>Ответ</summary>

```text
3
3
3
```

`var` имеет функциональную область видимости. Все callbacks обращаются к одной переменной после завершения цикла.

С `let` каждая итерация получает отдельное лексическое окружение:

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

Результат: `0`, `1`, `2`.

</details>

---

