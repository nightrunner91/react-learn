# TypeScript `infer` — выводи типы как детектив

`infer` — это ключевое слово TypeScript, которое позволяет **выводить типы внутри условных типов**. Звучит страшно, но на деле — это как детектив: ты смотришь на структуру типа и говоришь "ага, вот тут лежит число, а тут — строка". В этой статье разберём `infer` с самых основ, с аналогиями и пошаговыми примерами.

---

## Содержание

1. [Проблема, которую решает infer](#проблема-которую-решает-infer)
2. [Аналогия — infer как коробка](#аналогия--infer-как-коробка)
3. [Условные типы — база для infer](#условные-типы--база-для-infer)
4. [Первый пример infer](#первый-пример-infer)
5. [Как читать конструкции с infer](#как-читать-конструкции-с-infer)
6. [Извлечение типа из функции](#извлечение-типа-из-функции)
7. [Извлечение типа из массива](#извлечение-типа-из-массива)
8. [Извлечение типа из Promise](#извлечение-типа-из-promise)
9. [Несколько infer одновременно](#несколько-infer-одновременно)
10. [infer с ограничениями (TS 4.7+)](#infer-с-ограничениями-ts-47)
11. [Реальные примеры из практики](#реальные-примеры-из-практики)
12. [infer в React](#infer-в-react)
13. [Типичные ошибки](#типичные-ошибки)
14. [Шпаргалка](#шпаргалка)

---

## Проблема, которую решает infer

Представь, что у тебя есть тип `Promise<string>`. Ты хочешь написать utility-тип, который достанет `string` изнутри `Promise`. Как это сделать?

Без `infer` — никак. Ты не можешь "залезть" внутрь типа и посмотреть, что там.

```typescript
// Хочется что-то вроде:
type Unwrap<T> = ??? // как достать string из Promise<string>?
```

`infer` решает эту проблему — он позволяет TypeScript **самому вывести** тип, который лежит внутри.

---

## Аналогия — infer как коробка

Представь, что у тебя есть коробка с надписью `Promise<string>`. Ты не можешь открыть коробку, но можешь сказать:

> "Если эта коробка — Promise от чего-то (назовём это **R**), то верни мне **R**."

```typescript
type Unwrap<T> = T extends Promise<infer R> ? R : never;
```

Здесь `infer R` — это как сказать "положи содержимое коробки в переменную R". TypeScript сам разберётся, что внутри `Promise<string>` лежит `string`, и подставит `string` вместо `R`.

---

## Условные типы — база для infer

Прежде чем разбирать `infer`, нужно понять **условные типы**. Это как `if/else`, но для типов:

```typescript
type IsString<T> = T extends string ? "да" : "нет";

type A = IsString<string>;  // "да"
type B = IsString<number>;  // "нет"
```

Синтаксис: `A extends B ? X : Y`
- Если `A` подходит под шаблон `B`, то результат — `X`
- Иначе — `Y`

`infer` работает **только** внутри таких условных типов.

---

## Первый пример infer

Давай разберём самый простой пример:

```typescript
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = GetReturnType<() => string>;           // string
type B = GetReturnType<(x: number) => boolean>; // boolean
type C = GetReturnType<string>;                 // never
```

Разбираем по шагам:

1. `T extends (...args: any[]) => infer R` — "если T — это функция, которая возвращает что-то (назовём это R)"
2. `? R : never` — "тогда верни R, иначе верни never"

Когда мы передаём `() => string`:
- TypeScript видит, что это функция
- Видит, что она возвращает `string`
- Подставляет `string` вместо `infer R`
- Результат: `string`

Когда мы передаём `string`:
- TypeScript видит, что это не функция
- Условие не выполняется
- Результат: `never`

---

## Как читать конструкции с infer

Конструкции с `infer` выглядят страшно, но их легко разложить на части:

```typescript
T extends SomePattern<infer X> ? X : never
```

Читай так:
1. **"Если T подходит под шаблон `SomePattern<что-то>`..."**
2. **"...то назовём это 'что-то' буквой X"**
3. **"И вернём X"**

Пример:

```typescript
type ArrayElement<T> = T extends (infer E)[] ? E : never;
```

1. "Если T — это массив из чего-то..."
2. "...назовём это 'что-то' буквой E"
3. "Вернём E"

```typescript
type A = ArrayElement<string[]>; // string
type B = ArrayElement<number[]>; // number
```

---

## Извлечение типа из функции

### Возвращаемый тип

```typescript
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;

type A = ReturnOf<() => string>;              // string
type B = ReturnOf<(x: number) => boolean>;    // boolean
type C = ReturnOf<() => { id: number }>;      // { id: number }
```

### Первый параметр

```typescript
type FirstParam<T> = T extends (arg: infer P, ...rest: any[]) => any ? P : never;

type A = FirstParam<(x: string) => void>;     // string
type B = FirstParam<(n: number) => void>;     // number
```

### Все параметры как кортеж

```typescript
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

type A = Parameters<(x: string, y: number) => void>; // [string, number]
```

---

## Извлечение типа из массива

### Тип элемента массива

```typescript
type ElementType<T> = T extends (infer E)[] ? E : never;

type A = ElementType<string[]>;    // string
type B = ElementType<number[]>;    // number
type C = ElementType<[1, 2, 3]>;   // 1 | 2 | 3 (для кортежа)
```

### Первый элемент массива

```typescript
type First<T> = T extends [infer F, ...any[]] ? F : never;

type A = First<[string, number, boolean]>; // string
type B = First<[1, 2, 3]>;                 // 1
```

### Удалить первый элемент (Shift)

```typescript
type Shift<T extends any[]> = T extends [infer First, ...infer Rest] ? Rest : never;

type A = Shift<[1, 2, 3]>; // [2, 3]
type B = Shift<["a", "b"]>; // ["b"]
```

---

## Извлечение типа из Promise

```typescript
type Awaited<T> = T extends Promise<infer U> ? U : never;

type A = Awaited<Promise<string>>;  // string
type B = Awaited<Promise<number>>;  // number
```

Но что если Promise вложенный? `Promise<Promise<string>>`? Нужно рекурсия:

```typescript
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type A = Awaited<Promise<Promise<string>>>; // string
type B = Awaited<Promise<number>>;          // number
type C = Awaited<string>;                   // string (не Promise, верни как есть)
```

Разбираем `Awaited<Promise<Promise<string>>>`:
1. `T = Promise<Promise<string>>`, подходит под `Promise<infer U>`, где `U = Promise<string>`
2. Рекурсивно вызываем `Awaited<Promise<string>>`
3. `T = Promise<string>`, подходит под `Promise<infer U>`, где `U = string`
4. Рекурсивно вызываем `Awaited<string>`
5. `T = string`, не подходит под `Promise<...>`, возвращаем `string`

---

## Несколько infer одновременно

Можно использовать `infer` несколько раз в одном условном типе:

```typescript
type Shift<T extends any[]> = T extends [infer First, ...infer Rest] ? Rest : never;

type A = Shift<[1, 2, 3]>; // [2, 3]
```

Здесь:
- `infer First` — первый элемент (1)
- `infer Rest` — остаток ([2, 3])

### Извлечение из объекта

```typescript
type ExtractValue<T> = T extends { data: infer D } ? D : never;

type A = ExtractValue<{ data: string }>;  // string
type B = ExtractValue<{ data: number }>;  // number
type C = ExtractValue<{ id: number }>;    // never (нет поля data)
```

---

## infer с ограничениями (TS 4.7+)

Начиная с TypeScript 4.7, можно накладывать ограничения прямо на `infer`:

```typescript
type NumberOrString<T> = T extends infer U extends number | string ? U : never;

type A = NumberOrString<number>;  // number
type B = NumberOrString<string>;  // string
type C = NumberOrString<boolean>; // never (не подходит под ограничение)
```

Без этого синтаксиса пришлось бы писать более многословно:

```typescript
// До TS 4.7:
type NumberOrString<T> = T extends number | string ? T : never;

// С infer (TS 4.7+):
type NumberOrString<T> = T extends infer U extends number | string ? U : never;
```

В простых случаях разницы нет, но в сложных паттернах это позволяет точнее контролировать вывод типов.

---

## Реальные примеры из практики

### 1. Извлечение типа из React ref

```typescript
type RefType<T> = T extends React.RefObject<infer R> ? R : never;

type A = RefType<React.RefObject<HTMLInputElement>>; // HTMLInputElement
```

### 2. Извлечение типа из Observable (RxJS)

```typescript
type ObservableValue<T> = T extends Observable<infer V> ? V : never;

type A = ObservableValue<Observable<string>>; // string
```

### 3. Извлечение типа из конструктора

```typescript
type ConstructorType<T> = T extends new (...args: any[]) => infer R ? R : never;

class User {
  name: string = "";
}

type A = ConstructorType<typeof User>; // User
```

### 4. Deep Partial — рекурсивное making optional

```typescript
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

type User = {
  name: string;
  address: {
    street: string;
    city: string;
  };
};

type PartialUser = DeepPartial<User>;
// {
//   name?: string;
//   address?: {
//     street?: string;
//     city?: string;
//   };
// }
```

### 5. Извлечение типа из callback

```typescript
type CallbackParam<T> = T extends (callback: (value: infer V) => void) => void
  ? V
  : never;

type Fn = (callback: (value: string) => void) => void;
type A = CallbackParam<Fn>; // string
```

---

## infer в React

### Извлечение типа props из компонента

```typescript
type PropsOf<T> = T extends React.FC<infer P> ? P : never;

const MyComponent: React.FC<{ name: string }> = () => null;
type A = PropsOf<typeof MyComponent>; // { name: string }
```

### Извлечение типа из хука

```typescript
type UseStateType<T> = T extends [infer S, any] ? S : never;

type A = UseStateType<ReturnType<typeof useState<string>>>; // string
```

---

## Типичные ошибки

### 1. Использование infer вне условного типа

```typescript
// ОШИБКА: infer работает только внутри extends
type Bad<T> = infer R; // ❌ Error

// ПРАВИЛЬНО:
type Good<T> = T extends Promise<infer R> ? R : never; // ✅
```

### 2. Область действия infer

`infer` работает только в той ветке `extends`, где объявлен:

```typescript
type Test<T> = T extends Promise<infer R>
  ? R           // ✅ R доступна здесь
  : R;          // ❌ R недоступна здесь — будет ошибка
```

### 3. Конфликт имён

Если используешь несколько `infer` с одинаковым именем, будет конфликт:

```typescript
// ОШИБКА: нельзя использовать одно имя infer дважды в одном шаблоне
type Bad<T> = T extends [infer X, infer X] ? X : never; // ❌

// ПРАВИЛЬНО: используй разные имена
type Good<T> = T extends [infer First, infer Second] ? First | Second : never; // ✅
```

### 4. Забывать `never` в else-ветке

```typescript
// Плохо: если условие не выполнено, вернётся unknown
type Bad<T> = T extends Promise<infer R> ? R : unknown;

// Хорошо: явно указываем, что тип не найден
type Good<T> = T extends Promise<infer R> ? R : never;
```

---

## Шпаргалка

### Базовые паттерны

```typescript
// Извлечь возвращаемый тип функции
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;

// Извлечь первый параметр функции
type FirstParam<T> = T extends (arg: infer P, ...rest: any[]) => any ? P : never;

// Извлечь тип элемента массива
type ElementType<T> = T extends (infer E)[] ? E : never;

// Извлечь тип из Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

// Удалить первый элемент кортежа
type Shift<T extends any[]> = T extends [infer First, ...infer Rest] ? Rest : never;
```

### Как читать

```typescript
T extends SomePattern<infer X> ? X : never
```

Читай: *"Если T — это `SomePattern` от чего-то (назовём это X), то верни X. Иначе — never."*

### Ограничения

- `infer` работает **только** в `extends`-ветке условного типа
- Нельзя использовать `infer` как обычный generic-параметер
- Область действия `infer` — только тот условный тип, где он объявлен
- Нельзя использовать одно имя `infer` дважды в одном шаблоне

---

## Заключение

`infer` — это мощный инструмент для работы с типами. Он позволяет "заглянуть" внутрь типа и извлечь нужную информацию. Да, синтаксис поначалу выглядит пугающе, но если разбирать конструкции на части и использовать аналогии, всё становится понятным.

Главное — практиковаться. Попробуй написать свои utility-типы с `infer`, и через неделю ты будешь читать такие конструкции как обычную книгу.
