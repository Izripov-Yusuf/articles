# Callback рефы в React: что это такое и где можно применять

### Введение

При разработке у нас зачастую возникает необходимость прямого взаимодействия с DOM-элементами. Для такого случая React предоставляет нам механизм рефов (refs), который позволяет получать доступ к элементам после того, как они зарендерятся. Чаще всего используются обычные объектные рефы через useRef (обзовём их так), но также существует другой подход — callback refs. Этот метод даёт нам дополнительную гибкость и контроль над жизненным циклом элементов, позволяя выполнять необходимые нам специфические действия в точные моменты привязки и отвязки элементов. В этой статье я хочу объяснить, что такое callback refs, как они работают, показать проблемы при их использовании и примеры их использования.

### Что такое callback refs и как они работают

Callback refs дают более тонкий контроль над привязкой рефов по сравнению с объектными рефами. Рассмотрим, как они работают на деле:

1. **Монтирование:** Когда элемент монтируется в DOM, React вызывает функцию реф с самим DOM-элементом. Это позволяет вам выполнять действия с элементом сразу после его появления на странице.

2. **Размонтирование:** Когда элемент размонтируется, React вызывает функцию реф с null. Это даёт нам возможность очистить или отменить любые действия, связанные с элементом.

### Пример: отслеживание монтирования и размонтирования

```tsx
import React, { useCallback, useState } from 'react';

function MountUnmountTracker() {
  const [isVisible, setIsVisible] = useState(false);

  const handleRef = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Элемент монтирован:', node);
    } else {
      console.log('Элемент размонтирован');
    }
  }, []);

  return (
    <div>
      <button onClick={() => setIsVisible((prev) => !prev)}>
        {isVisible ? 'Скрыть' : 'Показать'} элемент
      </button>
      {isVisible && <div ref={handleRef}>Отслеживаемый элемент</div>}
    </div>
  );
}

export default MountUnmountTracker;
```

Каждый раз, когда мы переключаем видимость элемента, функция handleRef вызывается с соответствующим аргументом (node или null), позволяя отслеживать момент привязки и отвязки элемента.

## Частые проблемы и решения

### Проблема: повторные вызовы callback ref

Одна из частых проблем при использовании callback refs, это повторное создание функции рефа при каждом ре-рендере компонента. Из-за этого React думает, что у нас пришел новый реф, вызывает callback ref сначала с null, тем самым подчищая старый реф, а затем инициализирует новый, даже если сам наш элемент или компонент никак не изменились. В результате у нас могут возникнуть нежелаемые побочные эффекты.

**Пример проблемы**
Рассмотрим компонент Basic, который содержит кнопку для переключения видимости div с callback ref и кнопку производящую форс апдейт компонента:

```tsx
import React, { useState, useReducer } from 'react';

function Basic() {
  const [showDiv, setShowDiv] = useState(false);
  const [, forceRerender] = useReducer((v) => v + 1, 0);

  const toggleDiv = () => setShowDiv((prev) => !prev);

  const refCallback = (node: HTMLDivElement | null) => {
    console.log('div', node);
  };

  return (
    <div>
      <button onClick={toggleDiv}>Toggle Div</button>
      <button onClick={forceRerender}>Rerender</button>
      {showDiv && <div ref={refCallback}>Пример div</div>}
    </div>
  );
}

export default Basic;
```

Каждый раз при нажатии на кнопку Rerender, компонент перерисовывается, создавая новую функцию refCallback. Это приводит к вызову refCallback(null) и затем refCallback(node), не смотря на то, что наш элемент с рефом по сути никак не изменился. В консоли будут появляться сообщения с div и null поочерёдно снова и снова, чего мы конечно не хотели бы.

**Решение: Мемоизация callback ref с помощью useCallback**
Избежать это довольно легко, используйте useCallback для мемоизации функции. Это гарантирует, что функция останется неизменной между ре-рендерами, если её зависимости не изменились.

```tsx
import React, { useState, useCallback, useReducer } from 'react';

function Basic() {
  const [showDiv, setShowDiv] = useState(false);
  const [, forceRerender] = useReducer((v) => v + 1, 0);

  const toggleDiv = () => setShowDiv((prev) => !prev);

  const refCallback = useCallback((node: HTMLDivElement | null) => {
    console.log('div', node);
  }, []);

  return (
    <div>
      <button onClick={toggleDiv}>Toggle Div</button>
      <button onClick={forceRerender}>Rerender</button>
      {showDiv && <div ref={refCallback}>Пример div</div>}
    </div>
  );
}

export default Basic;
```

Теперь функция refCallback создаётся только один раз, при первом рендере, и не будет вызываться лишний раз при последующих ре-рендерах. Это предотвратит не нужные вызовы и улучшит производительность компонента.

## Порядок вызова callback refs, useLayoutEffect и useEffect

Перед тем как мы перейдём к тому, как использовать callback refs у нас в коде для решения проблем, давайте поймём, как callback refs взаимодействуют с хуками useEffect и useLayoutEffect, чтобы правильно организовать логику инициализации и очистки ресурсов.

### Порядок вызова

1. **callback ref:** Вызывается сразу после рендеринга DOM-элементов, **до** выполнения хуков эффекта.
2. **useLayoutEffect:** Выполняется после всех изменений DOM, **но до отрисовки**.
3. **useEffect:** Выполняется после отрисовки.

```tsx
import React, { useEffect, useLayoutEffect, useCallback } from 'react';

function WhenCalled() {
  const refCallback = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Callback ref вызван для div:', node);
    } else {
      console.log('Callback ref отвязал div');
    }
  }, []);

  useLayoutEffect(() => {
    console.log('useLayoutEffect вызван');
  }, []);

  useEffect(() => {
    console.log('useEffect вызван');
  }, []);

  return (
    <div>
      <div ref={refCallback}>Элемент для отслеживания</div>
    </div>
  );
}

export default WhenCalled;
```

**Вывод в консоль:**

1. "Callback ref вызван для div: [div элемент]"
2. "useLayoutEffect вызван"
3. "useEffect вызван"

Этот порядок показывает нам, что callback refs вызываются до хуков эффекта, что нужно учитывать при написании кода.

## Какие проблемы решают callback refs в коде

Для начала, давайте воспроизведём проблему с обычными объектными рефами, чтобы потом решить её через callback refs

```tsx
import { useCallback, useEffect, useRef, useState } from 'react';

interface ResizeObserverOptions {
  elemRef: React.RefObject<HTMLElement>;
  onResize: ResizeObserverCallback;
}

function useResizeObserver({ elemRef, onResize }: ResizeObserverOptions) {
  useEffect(() => {
    const element = elemRef.current;

    if (!element) {
      return;
    }

    const resizeObserver = new ResizeObserver(onResize);

    resizeObserver.observe(element);

    return () => {
      resizeObserver.unobserve(element);
    };
  }, [onResize, elemRef]);
}

export function UsageDom() {
  const [bool, setBool] = useState(false);
  const elemRef = useRef<HTMLDivElement>(null);

  const handleResize = useCallback((entries: ResizeObserverEntry[]) => {
    console.log('resize', entries);
  }, []);

  useResizeObserver({ elemRef, onResize: handleResize });

  const renderTestText = () => {
    if (bool) {
      return <p ref={elemRef}>Test text</p>;
    }

    return <div ref={elemRef}>Test div</div>;
  };

  return (
    <div style={{ width: '100%', textAlign: 'center' }}>
      <button onClick={() => setBool((v) => !v)}>Toggle</button>
      {renderTestText()}
    </div>
  );
}
```

Не будем тратить время на детальное объяснение того, что происходит в коде выше, если коротко, то мы отслеживаем размер нашего дива, либо параграфа через ResizeObserver.
По началу наш код отрабатывает правильно, при маунте мы получаем размер элемента, который отрендерился, при ресайзе в консоль также получаем его размер.

Проблемы возникают, когда мы начинаем тоглить наш стейт, тем самым меняя элемент, за которым мы следим.
Когда мы меняем стейт и элементы, за которыми следим, у нас перестаёт корректно работать ResizeObserver, он всё ещё продолжает следить за тем самым первым элементом, который уже удалён из DOM. Даже обратный тогл, который вроде бы возвращает нам самый первый элемент не помогает, т.к. подписка на новый элемент у нас просто не срабатывает

**Примечание:** Пример решения вышеописанной проблемы, которую мы далее решим через callback ref, скорее подходит для кода, который пишется в рамках библиотеки и который должен быть универсальным т.к. в реальном проекте мы можем решить всё это через комбинацию флагов, эффектов и т.п, в коде библиотеки мы не можем знать про конкретный компонент, его состояние и т.д. наш код обязан быть универсальным, в чём нам и поможет callback ref

```tsx
import { useCallback, useRef, useState } from 'react';

function useResizeObserver(onResize: ResizeObserverCallback) {
  const roRef = useRef<ResizeObserver | null>(null);

  const attachResizeObserver = useCallback(
    (element: HTMLElement) => {
      const resizeObserver = new ResizeObserver(onResize);
      resizeObserver.observe(element);
      roRef.current = resizeObserver;
    },
    [onResize]
  );

  const detachResizeObserver = useCallback(() => {
    roRef.current?.disconnect();
  }, []);

  const refCb = useCallback(
    (element: HTMLElement | null) => {
      if (element) {
        attachResizeObserver(element);
      } else {
        detachResizeObserver();
      }
    },
    [attachResizeObserver, detachResizeObserver]
  );

  return refCb;
}

export default function App() {
  const [bool, setBool] = useState(false);

  const handleResize = useCallback((entries: ResizeObserverEntry[]) => {
    console.log('resize', entries);
  }, []);

  const resizeRef = useResizeObserver(handleResize);

  const renderTestText = () => {
    if (bool) {
      return <p ref={resizeRef}>Test text</p>;
    }

    return <div ref={resizeRef}>Test div</div>;
  };

  return (
    <div style={{ width: '100%', textAlign: 'center' }}>
      <button onClick={() => setBool((v) => !v)}>Toggle</button>
      {renderTestText()}
    </div>
  );
}
```

Как видно, мы переписали наш хук useResizeObserver на callback ref, в который мы просто передаём коллбэк (замемоизированный!) который должен отрабатывать при ресайзе и теперь сколько бы мы не тоглили элементы, наш коллбэк с ресайзом будет отрабатывать, т.к. он навешивается на новые элементы и отвешивается от старых в необходимый нам момент времени благодаря callback ref. Самое главное в этом решении опять таки, это то, что разработчик, использующий наш хук не должен беспокоиться об этой логике навешивания/снятия обработчиков, мы инкапсулировали её у нас в хуке, разработчику остаётся лишь прокинуть коллбэк в наш хук и в рефы своих элементов

### Объединение нескольких рефов в один

Ещё один случай, где нам на помощь приходят callback ref

```tsx
import { useEffect, useRef } from 'react';
import { forwardRef, useCallback } from 'react';

interface InputProps {
  value?: string;
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
}

const Input = forwardRef(function Input(
  props: InputProps,
  ref: React.ForwardedRef<HTMLInputElement>
) {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    if (!inputRef.current) {
      return;
    }

    console.log(inputRef.current.getBoundingClientRect());
  }, []);

  return <input {...props} ref={ref} />;
});

export function UsageWithoutCombine() {
  const inputRef = useRef<HTMLInputElement | null>(null);

  const focus = () => {
    inputRef.current?.focus();
  };

  return (
    <div>
      <Input ref={inputRef} />
      <button onClick={focus}>Focus</button>
    </div>
  );
}
```

В примере выше у нас есть простой компонент инпута, на который мы навешиваем реф и получаем его из пропсов с помощью forwardRef.
Но что нам делать с inputRef в самом компоненте инпута? Может, мы хотим завязать на него другую логику, как в этом примере с getBoundingClientRect, но если мы заменим реф из пропсов на реф самого компонента, то у нас не сработает наш фокус. Как нам объединить наши два рефа?

Здесь нам опять приходят на помощь callback ref

```tsx
import { useEffect, useRef } from 'react';
import { forwardRef, useCallback } from 'react';

type RefItem<T> =
  | ((element: T | null) => void)
  | React.MutableRefObject<T | null>
  | null
  | undefined;

function useCombinedRef<T>(...refs: RefItem<T>[]) {
  const refCb = useCallback((element: T | null) => {
    refs.forEach((ref) => {
      if (!ref) {
        return;
      }

      if (typeof ref === 'function') {
        ref(element);
      } else {
        ref.current = element;
      }
    });
  }, refs);

  return refCb;
}

interface InputProps {
  value?: string;
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
}

const Input = forwardRef(function Input(
  props: InputProps,
  ref: React.ForwardedRef<HTMLInputElement>
) {
  const inputRef = useRef<HTMLInputElement>(null);
  const combinedInputRef = useCombinedRef(ref, inputRef);

  useEffect(() => {
    if (!inputRef.current) {
      return;
    }

    console.log(inputRef.current.getBoundingClientRect());
  }, []);

  return <input {...props} ref={combinedInputRef} />;
});

export function UsageWithCombine() {
  const inputRef = useRef<HTMLInputElement | null>(null);

  const focus = () => {
    inputRef.current?.focus();
  };

  return (
    <div>
      <Input ref={inputRef} />
      <button onClick={focus}>Focus</button>
    </div>
  );
}
```

**Объяснение:** Мы написали хук useCombinedRef, в который разработчик может передавать как обычные рефы, так и наши коллбэк рефы, ну и опционально мы даём ему возможность прокидывать null и undefined.

Сам хук useCombinedRef очень простой, это просто useCallback, обычная функция, в depth у него рефы, которые приходят в аргументах и при её вызове с конкретным элементом или null, если это null, то мы делаем return, если же реф это функция, то мы вызываем её с переданным нам параметром, если же это обычный реф, то мы просто обновляем его current.

С помощью useCombinedRef мы получаем 1 реф, но под капотом мы будем обновлять все нужные нам рефы. В примере выше например, у нас будет отрабатывать как getBoundingClientRect при маунте компонента Input, так и наша фокусировка на инпуте при клике на кнопку

## Что изменилось в React 19 в части callback refs

**Автоматическая очистка:** React теперь автоматически обрабатывает очистку callback refs при размонтировании элементов, что упрощает для нас задачу управления ресурсами. Вот какой пример приводит команда React у себя в документации

```jsx
<input
  ref={(ref) => {
    // ref created

    // NEW: return a cleanup function to reset
    // the ref when element is removed from DOM.
    return () => {
      // ref cleanup
    };
  }}
/>
```

Можете посмотреть на него более подробно по [ссылке](https://react.dev/blog/2024/12/05/react-19#cleanup-functions-for-refs), там в том числе написано про то, что скоро очистка рефа через null будет deprecate и останется только 1 способ очистки рефов

## Выбор между обычными refs и callback refs

- **Используйте обычные рефы** (useRef), когда вам нужен простой доступ к DOM-элементу или сохранение значения между рендерами без необходимости выполнять дополнительные действия при привязке или отвязке элемента.
- **Используйте callback refs**, когда требуется более тонкий контроль над жизненным циклом элемента, вы пишете универсальный код (у вас своя библиотека или пакет) или управление несколькими рефами.

## Заключение

Callback refs в React — это полезный инструмент, дающий разработчикам дополнительную гибкость и контроль над взаимодействием с DOM-элементами. Хотя в большинстве случаев стандартные объектные рефы через useRef полностью удовлетворяют наши с вами потребности, callback refs помогают в более сложных сценариях, которые мы обсудили выше
