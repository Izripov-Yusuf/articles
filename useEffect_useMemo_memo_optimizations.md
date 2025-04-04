# Взгляд с другой стороны на useMemo, useCallback и React.memo в React: когда их стоит использовать

## Введение

Оптимизация производительности React приложений — важная часть разработки, особенно когда речь идет о сложных интерфейсах. В основном разработчики лишь краем уха слышат о необходимости использования хуков useMemo, useCallback и React.memo для оптимизации кода. Но если бездумно использовать их, то можно даже навредить вашему приложению

В этой статье я попытаюсь разобрать, когда действительно стоит использовать useMemo, useCallback и React.memo, а когда их использование излишне. Мы изучим каждый из хуков, их влияние на рендеринг компонентов в React, а также рассмотрим практические примеры с подробными объяснениями работы каждого из хуков.

## Особенности рендеринга в React

Прежде чем углубиться в детали, важно понять, как именно работает рендеринг в React.

- **Компонент как функция:** В функциональном компоненте тело функции выполняется при каждом рендере. Это значит, что все переменные и функции, объявленные внутри компонента, будут пересоздаваться при каждом рендере.
- **Пересоздание функций и объектов:** Функции и объекты, созданные внутри компонента, будут "новыми" при каждом рендере (рекомендую почитать про сравнение по ссылке и по значению). Однако хуки, такие как useState и useEffect, сохраняют своё состояние между ре-рендерами благодаря внутренним механизмам React.

## Краткое описание хуков

### React.memo

React.memo — это **компонент высшего порядка (Higher Order Component, HOC)**, который мемоизирует функциональный компонент. Если пропсы компонента не изменились, React не сделает ре-рендер, он вернёт предыдущее состояние компонента.

**Особенности:**

- **Поверхностное сравнение (shallow equal):** React.memo сравнивает текущие и предыдущие пропсы с помощью Object.is, сравнивая их по ссылке.
- **Эффективен с пропсами примитивами:** Лучше всего работает, когда пропсы — примитивные типы (числа, строки, булевы значения).
- **Неэффективен с сложными объектами:** Если пропсы — объекты или функции, необходимо дополнительно использовать useMemo или useCallback для мемоизации этих пропсов.

### Примеры использования React.memo:

## Пример 1: Компонент с пропсами примитивами без React.memo

```jsx
function Display({ value }) {
  console.log('Рендер Display');
  return <div>{value}</div>;
}

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <div>
      <Display value="Статичный текст" />
      <button onClick={() => setCount(count + 1)}>Увеличить</button>
    </div>
  );
}
```

**Что происходит:** При каждом нажатии на кнопку, компонент Display делает ре-рендер, хотя его пропс value не изменился. Это происходит из-за того, что изменяется стейт, следовательно, родительский компонент App (в котором изменился стейт) перерисовывается, и его дочерние компоненты также должны перерисоваться.

## Пример 2: Компонент с примитивными пропсами, обёрнутый в React.memo

```jsx
const Display = React.memo(function Display({ value }) {
  console.log('Рендер Display');
  return <div>{value}</div>;
});

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <div>
      <Display value="Статичный текст" />
      <button onClick={() => setCount(count + 1)}>Увеличить</button>
    </div>
  );
}
```

**Что происходит:** Теперь компонент Display не перерисовывается при изменении стейта в App, так как его пропс value не изменился, и благодаря React.memo он избегает ненужного ре-рендеринга.

## Пример 3: Компонент с непримитивными пропсами, обёрнутый в React.memo

```jsx
const Display = React.memo(function Display({ data }) {
  console.log('Рендер Display');
  return <div>{data.value}</div>;
});

function App() {
  const [count, setCount] = React.useState(0);
  const data = { value: 'Статичный текст' };

  return (
    <div>
      <Display data={data} />
      <button onClick={() => setCount(count + 1)}>Увеличить</button>
    </div>
  );
}
```

**Что происходит:** Несмотря на то, что data не изменяется, Display перерисовывается при каждом ре-рендере App, потому что объект data пересоздаётся при каждом рендере, и его ссылка меняется (снова рекомендую почитать про сравнение по ссылке и по значению). React.memo видит, что пропс data изменился (ссылочно), и перерисовывает компонент.

**Как исправить:**

Использовать useMemo для мемоизации объекта data:

```jsx
const Display = React.memo(function Display({ data }) {
  console.log('Рендер Display');
  return <div>{data.value}</div>;
});

function App() {
  const [count, setCount] = React.useState(0);
  const data = React.useMemo(() => ({ value: 'Статичный текст' }), []);

  return (
    <div>
      <Display data={data} />
      <button onClick={() => setCount(count + 1)}>Увеличить</button>
    </div>
  );
}
```

Теперь data будет иметь стабильную ссылку между рендерами, и Display не будет перерисовываться без необходимости.

### useCallback

useCallback возвращает мемоизированную версию функции, которая сохраняется между рендерами, пока не изменятся указанные зависимости.

### Пример использования useCallback:

```jsx
const Button = React.memo(function Button({ onClick, label }) {
  console.log(`Рендер кнопки: ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

function Counter() {
  const [count, setCount] = React.useState(0);

  const increment = React.useCallback(() => setCount((c) => c + 1), []);

  return (
    <div>
      <h1>Счетчик: {count}</h1>
      <Button onClick={increment} label="Увеличить" />
    </div>
  );
}
```

### Что происходит:

- increment мемоизирован через useCallback, и его ссылка остаётся одинаковой между рендерами, пока зависимости не изменятся.
- Button обёрнут в React.memo, поэтому он не будет перерисовываться, пока его пропсы не изменятся.
- В данном случае, т.к. increment не использует переменных из внешнего окружения и они не изменяются, его ссылка остаётся стабильной.

### Обратите внимание:

- **Необходимо следить за массивом зависимостей, чтобы избежать проблем с устаревшими замыканиями (stale closure).**
- **Функция всё так же создаётся:** Несмотря на то, что ссылка на increment остаётся стабильной, сама функция пересоздаётся при каждом ре-рендере, но useCallback возвращает нам предыдущую версию, т.к. зависимости не изменились.

## Проблема с устаревшим замыканием

```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  const increment = React.useCallback(() => setCount(count + 1), []);

  return (
    <div>
      <h1>Счетчик: {count}</h1>
      <button onClick={increment}>Увеличить</button>
    </div>
  );
}
```

**Что происходит:**

- count не указан в зависимостях у useCallback.
- Из-за этого, increment всегда использует значение count, которое было при первом рендере.
- Это приводит к тому, что счётчик однажды увеличивается, но дальше не происходит увеличения, даже если мы кликнем 100500 раз.

**Решение:**

- Добавить count в зависимости (но в таком случае конечно теряется весь смысл его использования, но это только в нашем банальном примере так)

```jsx
const increment = React.useCallback(() => setCount(count + 1), [count]);
```

### useMemo

useMemo позволяет мемоизировать результат вычислений между ре-рендерами и пересчитывает его только тогда, когда изменятся зависимости.

### Пример использования useMemo

```jsx
function HeavyComputation({ num }) {
  const compute = (n) => {
    // Имитация тяжелых вычислений
    let result = 0;
    for (let i = 0; i < 1e7; i++) {
      result += n * Math.random();
    }
    return result;
  };

  const value = React.useMemo(() => compute(num), [num]);

  return <div>Результат вычислений: {value}</div>;
}

function App() {
  const [number] = React.useState(42);
  const [toggle, setToggle] = React.useState(false);

  return (
    <div>
      <button onClick={() => setToggle((t) => !t)}>Переключить</button>
      <HeavyComputation num={number} />
    </div>
  );
}
```

**Что происходит:**

- compute(num) выполняется только тогда, когда проп num изменяется.
- Это позволяет не делать лишние тяжелые вычисления при каждом ре-рендере App.

### Мемоизация объектов и массивов

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const data = React.useMemo(() => ({ value: 'Статичный текст' }), []);

  return (
    <div>
      <Display data={data} />
      <button onClick={() => setCount(count + 1)}>Увеличить</button>
    </div>
  );
}
```

- Теперь data имеет стабильную ссылку между ре-рендерами, и компоненты, зависящие от него, не будут перерисовываться без необходимости.

## Вопрос к читателям

Рассмотрим пример:

```jsx
function Button({ onClick, label }) {
  console.log(`Рендер кнопки: ${label}`);
  return <button onClick={onClick}>{label}</button>;
}

function Counter() {
  const [count, setCount] = React.useState(0);

  const increment = () => setCount(count + 1);

  return (
    <div>
      <h1>Счетчик: {count}</h1>
      <Button onClick={increment} label="Увеличить" />
    </div>
  );
}
```

Каждый раз при рендере Counter создаётся новая функция increment (для нас главное, что меняется ссылка на эту функцию), и Button получает новый проп onClick.

**Вопрос:** Как думаете, станет ли лучше производительность, если обернуть increment в useCallback и Button в React.memo? Какой из следующих вариантов будет более эффективным с точки зрения производительности?

1. Использовать useCallback для increment и обернуть Button в React.memo.
2. Оставить код без изменений.

**Подумайте над ответом, прежде чем читать дальше.**

## Объяснение

### Создание функций при каждом ре-рендере

- В функциональных компонентах все функции и объекты, объявленные внутри компонента, пересоздаются при КАЖДОМ рендере.
- Это значит, что ссылки на эти функции и объекты меняются при каждом ре-рендере, не смотря на то, что их содержимое остаётся без изменений.

## Влияние на дочерние компоненты

- Если дочерний компонент получает функцию или объект в качестве пропса и обёрнут в React.memo, изменение ссылки у какого-нибудь из пропсов всё равно приведёт к его перерисовке.
- Это вызовет ненужные перерисовки, если функции или объекты переданные в пропсах не мемоизированы.

### Ответ

На первый взгляд, оборачивание increment в useCallback и Button в React.memo должно предотвратить ненужные перерисовки Button. Но в данном случае выигрыш в производительности будет либо незначительным, либо его не будет вообще.

## Почему?

- **Простые компоненты:** Компонент Button очень простой, и время его рендеринга минимально. Оптимизация его ре-рендеров в этом случае не даёт какого-то заметного эффекта.
- **Мемоизации не бесплатны:** Использование useCallback и React.memo под капотом добавляет затраты на саму мемоизацию и сравнение пропсов между ре-рендерами.
- **Нужно следить за зависимостями:** В useCallback необходимо правильно указать зависимости, чтобы избежать проблем с устаревшими замыканиями.

**Резюмируя:** В этом примере, оборачивание increment в useCallback и использование React.memo для Button не даст нам значимого буста производительности, более того, это усложнит нам код. Поэтому оставить код в исходном виде лучше.

## Смысл оптимизации

**Главная мысль:** Надеюсь, вы заметили, что на протяжении всего текст я пытаюсь донести до вас вполне очевидную, но почему-то часто ускользающую мысль – оптимизация не бесплатна. Каждая оптимизация добавляет сложность со своей стороны и требует ресурсы. Ключевым является умение оценивать, а нужна ли нам здесь вообще эта оптимизация, не попадаем ли мы в ловушку преждевременной оптимизации.

- **Память и производительность:** Нельзя забывать, что мемоизация не бесплатна, она использует дополнительную память для хранения результатов и отслеживания зависимостей.
- **Сложность кода:** Повсеместное обёртывание функций, компонентов (тут место для холивара, что компонент и есть функция) и вычислений в хуки усложняет код и делает его менее читаемым.
- **Нужно измерять пользу от оптимизации:** Оптимизируйте только после того, как убедитесь что у вас есть проблемы с этим, используя инструменты профилирования.
- **Массив зависимостей и устаревшие замыкания:** Неправильное указание зависимостей может привести к багам. Всегда следите за тем, чтобы массив зависимостей был актуален.

Кстати, из последних новостей. В 19 версии разработчики React показали нам свой новый компайлер, который сам, под капотом занимается большей частью оптимизации, это вполне возможно сильно изменит привычный нам подход к оптимизации приложений

**Помните:** Главная цель — писать чистый, понятный и эффективный код. Оптимизации должны служить этой цели, а не препятствовать ей.
