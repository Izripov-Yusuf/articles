# Callback рефы в React: что это такое и с чем едят

### Введение

В разработке на React часто возникает необходимость прямого взаимодействия с DOM-элементами. Для этого React предоставляет механизм рефов (refs), который позволяет получать доступ к элементам после их рендеринга. Чаще всего разработчики используют объектные рефы через useRef, однако существует альтернативный подход — callback refs. Этот метод предоставляет дополнительную гибкость и контроль над жизненным циклом элементов, позволяя выполнять специфические действия в точные моменты привязки и отвязки элементов. В этой статье мы подробно рассмотрим, что такое callback refs, когда их стоит использовать, а также разберёмся с практическими примерами и возможными подводными камнями при их применении.

### Что такое Callback Refs?

Callback refs — это функция, которую вы передаёте в атрибут ref элемента или компонента. В отличие от объектных рефов, React вызывает эту функцию дважды:

1. **При монтировании элемента:** функция вызывается с самим DOM-элементом или экземпляром компонента.
2. **При размонтировании элемента:** функция вызывается с null.

Этот механизм позволяет выполнять определённые действия именно в тот момент, когда элемент появляется или исчезает из DOM, что открывает дополнительные возможности для управления жизненным циклом элемента.

### Простой пример Callback Ref

Рассмотрим базовый пример функционального компонента, который устанавливает фокус на инпут при его монтировании:

```tsx
import React, { useCallback } from 'react';

function InputWithCallbackRef() {
  const handleRef = useCallback((element: HTMLInputElement | null) => {
    if (element) {
      element.focus();
      console.log('Инпут сфокусирован:', element);
    } else {
      console.log('Инпут размонтирован');
    }
  }, []);

  return <input ref={handleRef} placeholder="Автоматический фокус" />;
}

export default InputWithCallbackRef;
```

В этом примере функция handleRef автоматически устанавливает фокус на инпут при его монтировании и выводит сообщение в консоль при размонтировании. Использование useCallback гарантирует, что функция handleRef не будет пересоздаваться при каждом рендере, что предотвращает лишние вызовы.

### Преимущества Callback Refs

1. ### Гибкость и Контроль

   Callback refs предоставляют более точный контроль над моментами привязки и отвязки рефа. Это особенно полезно, когда необходимо выполнить определённые действия сразу после того, как элемент появился в DOM или перед его удалением. Например, можно инициализировать стороннюю библиотеку или подписаться на события только тогда, когда элемент доступен.

2. ### Интеграция со Сторонними Библиотеками

   При работе со сторонними библиотеками, которые требуют непосредственного взаимодействия с DOM-элементами (например, инициализация слайдеров, графиков или карт), callback refs позволяют точно определить момент инициализации и очистки ресурсов. Это обеспечивает более надёжную интеграцию и предотвращает утечки памяти или некорректное поведение компонентов.

3. ### Управление Множественными Refs
   Если вам нужно управлять множеством рефов динамически, callback refs облегчают эту задачу. Вместо создания отдельного объекта рефа для каждого элемента, можно использовать одну функцию-реф для всех элементов, что упрощает код и улучшает его читаемость.

### Частые Проблемы и Решения

Проблема: Повторные Вызовы Callback Ref
Одной из распространённых проблем при использовании callback refs является повторное создание функции-рефа при каждом рендере компонента. Это приводит к тому, что React вызывает callback ref с null и затем с новым элементом, даже если сам элемент не изменился. В результате могут возникать нежелательные побочные эффекты, такие как лишние операции или сбои в логике.

Пример Проблемы
Рассмотрим компонент Basic, который содержит кнопку для переключения видимости div с callback ref:

```tsx
import React, { useState } from 'react';

function Basic() {
  const [showDiv, setShowDiv] = useState(false);

  const toggleDiv = () => setShowDiv((prev) => !prev);

  const refCallback = (node: HTMLDivElement | null) => {
    if (node) {
      console.log('Привязали div:', node);
    } else {
      console.log('Отвязали div');
    }
  };

  return (
    <div>
      <button onClick={toggleDiv}>Toggle Div</button>
      {showDiv && <div ref={refCallback}>Пример div</div>}
    </div>
  );
}

export default Basic;
```

Каждый раз при нажатии на кнопку Toggle Div, компонент перерисовывается, создавая новую функцию refCallback. Это приводит к вызову refCallback(null) и затем refCallback(node), даже если элемент остаётся тем же самым. В консоли будут появляться сообщения об отвязке и привязке div снова и снова, что может быть нежелательно.

**Решение: Мемоизация Callback Ref с помощью useCallback**
Чтобы избежать повторного создания функции-рефа при каждом рендере, используйте useCallback для мемоизации функции. Это гарантирует, что функция останется неизменной между рендерами, если её зависимости не изменились.

```tsx
import React, { useState, useCallback } from 'react';

function Basic() {
  const [showDiv, setShowDiv] = useState(false);

  const toggleDiv = () => setShowDiv((prev) => !prev);

  const refCallback = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Привязали div:', node);
    } else {
      console.log('Отвязали div');
    }
  }, []);

  return (
    <div>
      <button onClick={toggleDiv}>Toggle Div</button>
      {showDiv && <div ref={refCallback}>Пример div</div>}
    </div>
  );
}

export default Basic;
```

Теперь функция refCallback создаётся только один раз, при первом рендере, и не будет вызываться лишний раз при последующих обновлениях состояния. Это предотвращает нежелательные вызовы и улучшает производительность компонента.

### Проблема: Управление Множественными Refs

При работе с динамическими списками элементов может возникнуть необходимость управлять множеством рефов. Без правильной организации это может привести к сложности и путанице в коде.

**Решение: Использование Кастомного Хука useCombinedRefs**
Создайте кастомный хук, который объединяет несколько рефов в один callback ref, облегчая управление ими.

```tsx
import React, { useRef, useCallback } from 'react';

function useCombinedRefs<T>(...refs: React.Ref<T>[]) {
  return useCallback(
    (node: T | null) => {
      refs.forEach((ref) => {
        if (!ref) return;
        if (typeof ref === 'function') {
          ref(node);
        } else {
          (ref as React.MutableRefObject<T | null>).current = node;
        }
      });
    },
    [refs]
  );
}

export default useCombinedRefs;
```

Этот хук позволяет одновременно использовать несколько рефов без необходимости создавать отдельные функции для каждого из них. Применение useCombinedRefs упрощает код и делает его более управляемым.

## Практические Примеры

### Пример 1: Простой Компонент с Callback Ref

Компонент Basic демонстрирует базовое использование callback ref для отслеживания монтирования и размонтирования div.

```tsx
import React, { useState, useCallback } from 'react';

function Basic() {
  const [showDiv, setShowDiv] = useState(false);

  const toggleDiv = () => setShowDiv((prev) => !prev);

  const refCallback = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Привязали div:', node);
    } else {
      console.log('Отвязали div');
    }
  }, []);

  return (
    <div>
      <button onClick={toggleDiv}>Toggle Div</button>
      {showDiv && <div ref={refCallback}>Пример div</div>}
    </div>
  );
}

export default Basic;
```

При нажатии на кнопку Toggle Div происходит монтирование и размонтирование div, что приводит к вызовам функции refCallback с соответствующими аргументами.

### Пример 2: Проблема с Повторными Вызовами Callback Ref

Компонент NullProblem показывает, как повторные вызовы callback refs могут приводить к проблемам при частых рендерах компонента.

```tsx
import React, { useState, useCallback } from 'react';

function NullProblem() {
  const [count, setCount] = useState(0);

  const handleRef = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Привязали div:', node);
    } else {
      console.log('Отвязали div');
    }
  }, []);

  const increment = () => setCount((prev) => prev + 1);

  return (
    <div>
      <button onClick={increment}>Увеличить счётчик</button>
      <div ref={handleRef}>Счётчик: {count}</div>
    </div>
  );
}

export default NullProblem;
```

Каждое нажатие на кнопку Увеличить счётчик приводит к перерисовке компонента и повторным вызовам handleRef с null и вновь с div, даже если элемент остаётся тем же.

### Пример 3: Объединение Refs с помощью useCombinedRefs

Компонент UsageCombine демонстрирует, как можно объединить несколько рефов в один с помощью кастомного хука useCombinedRefs.

```tsx
import React, { useRef } from 'react';
import useCombinedRefs from './useCombinedRefs';

function UsageCombine() {
  const localRef = useRef<HTMLDivElement | null>(null);
  const externalRef = useRef<HTMLDivElement | null>(null);

  const combinedRef = useCombinedRefs(localRef, externalRef);

  return <div ref={combinedRef}>Элемент с объединёнными рефами</div>;
}

export default UsageCombine;
```

Этот подход позволяет одновременно использовать localRef и externalRef для одного DOM-элемента, обеспечивая гибкость в управлении рефами.

### Пример 4: Измерение Размеров DOM-Элемента

Компонент UsageDom показывает, как callback ref можно использовать для измерения размеров элемента сразу после его монтирования.

```tsx
import React, { useCallback, useState } from 'react';

function UsageDom() {
  const [size, setSize] = useState({ width: 0, height: 0 });

  const refCallback = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      const { offsetWidth, offsetHeight } = node;
      setSize({ width: offsetWidth, height: offsetHeight });
      console.log(`Размеры div: ${offsetWidth}x${offsetHeight}`);
    }
  }, []);

  return (
    <div>
      <div
        ref={refCallback}
        style={{ width: '200px', height: '100px', background: 'lightblue' }}
      >
        Измеряемый div
      </div>
      <p>
        Ширина: {size.width}px, Высота: {size.height}px
      </p>
    </div>
  );
}

export default UsageDom;
```

При монтировании div вызывается refCallback, который измеряет его размеры и обновляет состояние size.

### Пример 5: Порядок Вызовов Callback Refs и Эффектов

Компонент WhenCalled демонстрирует порядок, в котором React вызывает callback refs относительно хуков эффектов.

```tsx
import React, { useEffect, useCallback } from 'react';

function WhenCalled() {
  const refCallback = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      console.log('Callback ref вызван для div:', node);
    } else {
      console.log('Callback ref отвязал div');
    }
  }, []);

  useEffect(() => {
    console.log('useEffect в родительском компоненте');
  }, []);

  return (
    <div>
      <div ref={refCallback}>Элемент для отслеживания</div>
    </div>
  );
}

export default WhenCalled;
```

Этот пример показывает, что callback ref вызывается до выполнения хуков эффектов, что важно учитывать при планировании логики инициализации и очистки ресурсов.

## Важные Моменты при Использовании Callback Refs

Мемоизация Функций-рефов
Чтобы избежать повторных вызовов и нежелательных побочных эффектов, всегда мемоизируйте функции-рефы с помощью useCallback. Это гарантирует, что функция остаётся неизменной между рендерами, если её зависимости не изменились.

```tsx
const refCallback = useCallback((node: HTMLElement | null) => {
  // Логика при привязке и отвязке
}, []);
```

### Избегайте Хранения Элементов в Состоянии

Хотя технически возможно сохранять элементы в состоянии компонента, это может привести к нежелательным ререндерингам. Лучше использовать useRef для хранения ссылок на элементы, если вам нужно обращаться к ним вне функции-рефа.

```tsx
const elementRef = useRef<HTMLDivElement | null>(null);

const handleRef = useCallback((node: HTMLDivElement | null) => {
  if (node) {
    elementRef.current = node;
    // Дополнительная логика
  } else {
    elementRef.current = null;
    // Очистка логики
  }
}, []);
```

### Совместимость с React.memo и HOC

Если вы передаёте функцию-реф как проп в дочерний компонент, который обёрнут в React.memo, убедитесь, что функция-реф мемоизирована. Иначе, каждый раз при изменении родительского компонента, дочерний компонент будет ререндериться из-за изменения ссылки на функцию-реф.

```tsx
const ParentComponent = () => {
  const refCallback = useCallback((node: HTMLDivElement | null) => {
    // Логика рефа
  }, []);

  return <ChildComponent ref={refCallback} />;
};

const ChildComponent = React.memo(
  ({ ref }: { ref: React.Ref<HTMLDivElement> }) => {
    return <div ref={ref}>Дочерний компонент</div>;
  }
);
```

## Заключение

Callback refs в React — это мощный инструмент, предоставляющий разработчикам дополнительную гибкость и контроль над взаимодействием с DOM-элементами. Хотя в большинстве случаев стандартные объектные рефы через useRef полностью удовлетворяют потребности, callback refs становятся незаменимыми в более сложных сценариях, таких как интеграция со сторонними библиотеками или динамическое управление множественными элементами.

**Ключевые Выводы:**

1. Гибкость: Callback refs позволяют выполнять действия точно в момент привязки и отвязки элемента.
2. Контроль: Идеально подходят для ситуаций, требующих точной инициализации и очистки ресурсов.
3. Интеграция: Облегчают работу со сторонними библиотеками, требующими прямого доступа к DOM.
4. Мемоизация: Использование useCallback для мемоизации функций-рефов помогает избежать лишних вызовов и побочных эффектов.
5. Управление множественными рефами: Кастомные хуки, такие как useCombinedRefs, упрощают управление несколькими рефами одновременно.

**Советы по Использованию:**

- Мемоизируйте функции-рефы: Это поможет избежать повторных вызовов и нежелательных побочных эффектов.
- Оценивайте необходимость: Не усложняйте код без необходимости. В большинстве случаев useRef справляется отлично.
- Используйте кастомные хуки: Для объединения рефов или других сложных сценариев создавайте кастомные хуки, чтобы сделать код более читаемым и управляемым.
- Понимайте жизненный цикл: Важно понимать, когда именно React вызывает callback refs, чтобы правильно управлять логикой инициализации и очистки.
