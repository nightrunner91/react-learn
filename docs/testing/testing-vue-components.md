# Тестирование Vue-компонентов с Vue Test Utils

## Содержание

1. [Что такое Vue Test Utils](#что-такое-vue-test-utils)
2. [Философия: тестируйте как пользователь](#философия-тестируйте-как-пользователь)
3. [Установка и настройка](#установка-и-настройка)
4. [mount — рендеринг компонентов](#mount--рендеринг-компонентов)
5. [find / findAll — поиск элементов](#find--findall--поиск-элементов)
6. [trigger — события](#trigger--события)
7. [Тестирование пропсов](#тестирование-пропсов)
8. [Тестирование emits](#тестирование-emits)
9. [Тестирование слотов](#тестирование-слотов)
10. [Тестирование v-model](#тестирование-v-model)
11. [Тестирование асинхронных компонентов](#тестирование-асинхронных-компонентов)
12. [Тестирование composables](#тестирование-composables)
13. [Тестирование с Pinia](#тестирование-с-pinia)
14. [Тестирование с Vue Router](#тестирование-с-vue-router)
15. [Тестирование provide / inject](#тестирование-provide--inject)
16. [Кастомные mount-обёртки](#кастомные-mount-обёртки)
17. [Лучшие практики](#лучшие-практики)
18. [Антипаттерны](#антипаттерны)

---

## Что такое Vue Test Utils

**Vue Test Utils (VTU)** — официальная библиотека для тестирования Vue-компонентов. Она предоставляет API для монтирования компонентов в изолированном окружении, поиска элементов, имитации событий и взаимодействия с инстансом компонента.

### Ключевые отличия от React Testing Library

| Характеристика | Vue Test Utils | React Testing Library |
|---|---|---|
| **Философия** | Тестировать поведение, но иметь доступ к инстансу | Тестировать только поведение пользователя |
| **Доступ к инстансу** | ✅ Да (`wrapper.vm`) | ❌ Нет |
| **Поиск элементов** | `find`, `findAll`, `findComponent` | `screen.getBy*`, `screen.queryBy*` |
| **События** | `trigger` | `userEvent`, `fireEvent` |
| **Компоненты-обёртки** | `global.plugins`, `global.provide` | `wrapper` в `renderHook`, кастомный `render` |
| **Доступ к emits** | `wrapper.emitted()` | Моки колбэков |

> 💡 Vue Test Utils даёт больше доступа к внутреннему состоянию компонента, чем React Testing Library. Это мощный инструмент, но им нужно пользоваться осторожно: тесты, завязанные на `wrapper.vm`, становятся хрупкими при рефакторинге.

---

## Философия: тестируйте как пользователь

Главный принцип тестирования компонентов одинаков для Vue и React: **тестируйте то, что видит и делает пользователь, а не внутреннее устройство компонента**.

```vue
<!-- Counter.vue -->
<template>
  <button @click="increment">Count: {{ count }}</button>
</template>

<script setup>
import { ref } from "vue";

const count = ref(0);

function increment() {
  count.value++;
}
</script>
```

```ts
// ❌ Тестирует внутреннее состояние — хрупкий тест
it("increments internal count", () => {
  const wrapper = mount(Counter);
  wrapper.vm.count++;
  expect(wrapper.vm.count).toBe(1);
});

// ✅ Тестирует поведение пользователя — устойчивый тест
it("increments count on button click", async () => {
  const wrapper = mount(Counter);
  await wrapper.find("button").trigger("click");
  expect(wrapper.find("button").text()).toBe("Count: 1");
});
```

### Что проверяет пользователь

Пользователь не знает о `ref`, `reactive`, `computed`, `emits`. Он видит:

- **Текст** на странице
- **Кнопки** и интерактивные элементы
- **Поля ввода** и их значения
- **Изменения** после взаимодействия

Тесты должны проверять то же самое.

---

## Установка и настройка

Для тестирования Vue-компонентов нужны три пакета: сам Vue Test Utils, Vitest как тестовый раннер и DOM-окружение (`jsdom` или `happy-dom`).

```bash
npm install -D @vue/test-utils vitest jsdom
```

### Конфигурация Vitest для Vue

Vitest использует Vite, поэтому для `.vue` файлов нужен `@vitejs/plugin-vue`. Окружение `jsdom` эмулирует браузерный DOM в Node.js.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: "jsdom",
    globals: true,
  },
});
```

### happy-dom vs jsdom

| Характеристика | jsdom | happy-dom |
|---|---|---|
| **Скорость** | Медленнее | Быстрее |
| **Совместимость** | Выше | Ниже, но достаточна для большинства тестов |
| **Ресурсы** | Больше | Меньше |
| **Когда использовать** | Сложные DOM-операции, сторонние библиотеки | Обычные компонентные тесты |

```bash
npm install -D happy-dom
```

```ts
// vitest.config.ts
export default defineConfig({
  plugins: [vue()],
  test: {
    environment: "happy-dom",
  },
});
```

---

## mount — рендеринг компонентов

`mount` — главная функция Vue Test Utils. Она создаёт инстанс компонента, рендерит его в виртуальный DOM и возвращает `wrapper` — объект с методами для поиска элементов и доступа к компоненту.

### Базовый mount

```ts
import { mount } from "@vue/test-utils";
import Counter from "./Counter.vue";

it("renders counter", () => {
  const wrapper = mount(Counter);
  expect(wrapper.find("button").exists()).toBe(true);
});
```

### Что возвращает mount

```ts
const wrapper = mount(MyComponent, {
  props: { title: "Hello" },
});

wrapper.element;      // Корневой DOM-элемент
wrapper.html();       // HTML-строка компонента
wrapper.text();       // Текстовое содержимое
wrapper.vm;           // Инстанс компонента (осторожно)
wrapper.find("...");  // Поиск элемента
wrapper.findComponent(Component); // Поиск дочернего компонента
wrapper.emitted();    // События, которые эмитнул компонент
wrapper.unmount();    // Размонтировать компонент
```

### Передача пропсов

```ts
const wrapper = mount(Greeting, {
  props: {
    name: "Alice",
  },
});
```

### Глобальные плагины и провайдеры

```ts
const wrapper = mount(MyComponent, {
  global: {
    plugins: [pinia],
    provide: {
      theme: "dark",
    },
    stubs: ["FontAwesomeIcon"],
  },
});
```

---

## find / findAll — поиск элементов

VTU предоставляет несколько способов поиска элементов в DOM. Предпочитайте семантические селекторы: по тегу, роли, тексту. CSS-классы — последний резерв.

### Типы селекторов

| Метод | Описание | Пример |
|---|---|---|
| `find("button")` | По CSS-селектору | `wrapper.find("button")` |
| `find("[data-testid='submit']")` | По data-testid | `wrapper.find('[data-testid="submit"]')` |
| `findComponent(Component)` | По компоненту | `wrapper.findComponent(Child)` |
| `findAll("li")` | Все совпадающие элементы | `wrapper.findAll("li")` |

### getByRole через DOM-методы

VTU не имеет встроенного `getByRole`, но можно использовать `wrapper.find` с селекторами ролей или подключить `@testing-library/vue`.

```bash
npm install -D @testing-library/vue
```

```ts
import { render, screen } from "@testing-library/vue";

it("finds button by role", () => {
  render(Counter);
  expect(screen.getByRole("button", { name: /count/i })).toBeInTheDocument();
});
```

> 💡 `@testing-library/vue` — это обёртка над Vue Test Utils с философией RTL. Если вы привыкли к React Testing Library, переход будет плавным.

### Проверка наличия и отсутствия

```ts
// Элемент есть в DOM
expect(wrapper.find(".error").exists()).toBe(true);

// Элемента нет
expect(wrapper.find(".error").exists()).toBe(false);

// Текст содержится
expect(wrapper.text()).toContain("Error message");

// Количество элементов
expect(wrapper.findAll("li")).toHaveLength(3);
```

---

## trigger — события

`trigger` имитирует DOM-события. Он возвращает Promise, поэтому события нужно `await`. В отличие от `fireEvent` в RTL, `trigger` отправляет одно событие — для полной имитации пользовательского ввода иногда нужно отправлять несколько событий последовательно.

### Клик по кнопке

```ts
it("increments count on click", async () => {
  const wrapper = mount(Counter);
  await wrapper.find("button").trigger("click");
  expect(wrapper.find("button").text()).toBe("Count: 1");
});
```

### Ввод текста

```ts
it("updates input value", async () => {
  const wrapper = mount(SearchInput);
  const input = wrapper.find("input");

  await input.setValue("hello");

  expect(input.element.value).toBe("hello");
});
```

> 💡 `setValue` — удобный метод VTU, который устанавливает `value` и триггерит `input` событие. Это аналог `userEvent.type` в React Testing Library.

### Отправка формы

```ts
it("submits form", async () => {
  const wrapper = mount(LoginForm);
  await wrapper.find("input[type='email']").setValue("user@example.com");
  await wrapper.find("form").trigger("submit");

  expect(wrapper.emitted("submit")).toBeTruthy();
});
```

### Чекбокс

```ts
it("toggles checkbox", async () => {
  const wrapper = mount(AgreementCheckbox);
  const checkbox = wrapper.find("input[type='checkbox']");

  expect(checkbox.element.checked).toBe(false);

  await checkbox.setValue(true);

  expect(checkbox.element.checked).toBe(true);
});
```

### Выбор из select

```ts
it("selects option", async () => {
  const wrapper = mount(CountrySelect);
  const select = wrapper.find("select");

  await select.setValue("us");

  expect(select.element.value).toBe("us");
});
```

---

## Тестирование пропсов

Пропсы — основной способ передачи данных в компонент. Тесты должны проверять, как компонент реагирует на разные пропсы: рендеринг текста, условные классы, disabled-состояние.

```vue
<!-- Alert.vue -->
<template>
  <div :class="['alert', `alert--${type}`]">
    {{ message }}
  </div>
</template>

<script setup>
defineProps({
  message: String,
  type: {
    type: String,
    default: "info",
  },
});
</script>
```

```ts
it("renders message prop", () => {
  const wrapper = mount(Alert, {
    props: { message: "Hello" },
  });
  expect(wrapper.text()).toBe("Hello");
});

it("applies type class", () => {
  const wrapper = mount(Alert, {
    props: { message: "Error", type: "error" },
  });
  expect(wrapper.find(".alert--error").exists()).toBe(true);
});
```

### Обновление пропсов

```ts
it("updates when props change", async () => {
  const wrapper = mount(Greeting, {
    props: { name: "Alice" },
  });

  expect(wrapper.text()).toContain("Alice");

  await wrapper.setProps({ name: "Bob" });

  expect(wrapper.text()).toContain("Bob");
});
```

---

## Тестирование emits

Компоненты Vue общаются с родителями через события (`emits`). VTU отслеживает все события через `wrapper.emitted()`.

```vue
<!-- SearchForm.vue -->
<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="query" />
    <button type="submit">Search</button>
  </form>
</template>

<script setup>
import { ref } from "vue";

const query = ref("");
const emit = defineEmits(["search"]);

function handleSubmit() {
  emit("search", query.value);
}
</script>
```

```ts
it("emits search event with query", async () => {
  const wrapper = mount(SearchForm);

  await wrapper.find("input").setValue("react");
  await wrapper.find("form").trigger("submit");

  expect(wrapper.emitted("search")).toHaveLength(1);
  expect(wrapper.emitted("search")[0]).toEqual(["react"]);
});
```

### Проверка отсутствия события

```ts
it("does not emit when input is empty", async () => {
  const wrapper = mount(SearchForm);
  await wrapper.find("form").trigger("submit");

  expect(wrapper.emitted("search")).toBeFalsy();
});
```

---

## Тестирование слотов

Слоты — мощный механизм композиции во Vue. Тесты должны проверять, что компонент корректно рендерит переданный контент.

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <div class="card__header">
      <slot name="header" />
    </div>
    <div class="card__body">
      <slot />
    </div>
  </div>
</template>
```

```ts
it("renders default and named slots", () => {
  const wrapper = mount(Card, {
    slots: {
      default: "Card content",
      header: "Card title",
    },
  });

  expect(wrapper.text()).toContain("Card title");
  expect(wrapper.text()).toContain("Card content");
});
```

### Слоты с пропсами (scoped slots)

```ts
it("renders scoped slot", () => {
  const wrapper = mount(DataTable, {
    slots: {
      row: `<template #row="{ item }">{{ item.name }}</template>`,
    },
  });

  expect(wrapper.text()).toContain("Alice");
});
```

---

## Тестирование v-model

`v-model` — синтаксический сахар для пропса `modelValue` и события `update:modelValue`. В тестах нужно передать `modelValue` и проверить, что компонент эмитит `update:modelValue`.

```vue
<!-- TextInput.vue -->
<template>
  <input
    :value="modelValue"
    @input="$emit('update:modelValue', $event.target.value)"
  />
</template>

<script setup>
defineProps({
  modelValue: String,
});

defineEmits(["update:modelValue"]);
</script>
```

```ts
it("emits update:modelValue on input", async () => {
  const wrapper = mount(TextInput, {
    props: {
      modelValue: "",
      "onUpdate:modelValue": (value) => wrapper.setProps({ modelValue: value }),
    },
  });

  await wrapper.find("input").setValue("hello");

  expect(wrapper.emitted("update:modelValue")).toHaveLength(1);
  expect(wrapper.emitted("update:modelValue")[0]).toEqual(["hello"]);
});
```

> 💡 В Vue 3 `v-model` можно кастомизировать через `defineModel`, что упрощает тестирование — не нужно вручную связывать пропс и событие.

---

## Тестирование асинхронных компонентов

Асинхронные операции — запросы к API, таймеры, анимации — требуют ожидания. В Vue Test Utils для этого используется `flushPromises`.

```ts
import { flushPromises } from "@vue/test-utils";
```

### Компонент с загрузкой данных

```vue
<!-- UserList.vue -->
<template>
  <div>
    <div v-if="loading">Loading...</div>
    <ul v-else>
      <li v-for="user in users" :key="user.id">{{ user.name }}</li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";

const users = ref([]);
const loading = ref(true);

onMounted(async () => {
  const response = await fetch("/api/users");
  users.value = await response.json();
  loading.value = false;
});
</script>
```

### Тест с MSW

```ts
it("displays users after loading", async () => {
  server.use(
    http.get("/api/users", () =>
      HttpResponse.json([
        { id: 1, name: "Alice" },
        { id: 2, name: "Bob" },
      ])
    )
  );

  const wrapper = mount(UserList);

  expect(wrapper.text()).toContain("Loading...");

  await flushPromises();

  expect(wrapper.text()).toContain("Alice");
  expect(wrapper.text()).toContain("Bob");
  expect(wrapper.text()).not.toContain("Loading...");
});
```

### Ожидание через find и retry

```ts
it("waits for element", async () => {
  const wrapper = mount(AsyncComponent);

  await vi.waitFor(() => {
    expect(wrapper.find(".loaded").exists()).toBe(true);
  });
});
```

---

## Тестирование composables

Composables — аналог кастомных хуков в React. Их можно тестировать напрямую через `mount` в тестовом компоненте или с помощью `@vue/test-utils` + вспомогательного компонента.

### Через тестовый компонент

```ts
import { mount } from "@vue/test-utils";
import { defineComponent } from "vue";
import { useCounter } from "./useCounter";

it("useCounter increments", async () => {
  const TestComponent = defineComponent({
    setup() {
      return useCounter();
    },
    template: "<button @click=\"increment\">{{ count }}</button>",
  });

  const wrapper = mount(TestComponent);

  expect(wrapper.text()).toBe("0");

  await wrapper.find("button").trigger("click");

  expect(wrapper.text()).toBe("1");
});
```

### Через @vue/composition-api-test-utils

В небольших проектах удобно создавать вспомогательный компонент прямо в тесте. Для сложных composables лучше выносить логику в чистые функции и тестировать их отдельно.

---

## Тестирование с Pinia

Pinia — стандарт управления состоянием во Vue 3. Для тестирования нужно создать тестовый store и передать его в `global.plugins`.

```ts
import { setActivePinia, createPinia } from "pinia";
import { useCartStore } from "./stores/cart";

beforeEach(() => {
  setActivePinia(createPinia());
});

it("adds item to cart", () => {
  const cart = useCartStore();
  cart.addItem({ id: 1, name: "Book", price: 100 });

  expect(cart.items).toHaveLength(1);
  expect(cart.total).toBe(100);
});
```

### Компонент с Pinia

```ts
import { createPinia } from "pinia";

it("displays cart total", () => {
  const pinia = createPinia();
  const wrapper = mount(CartBadge, {
    global: {
      plugins: [pinia],
    },
  });

  const cart = useCartStore();
  cart.addItem({ id: 1, name: "Book", price: 100 });

  expect(wrapper.text()).toContain("100");
});
```

### Мокирование store

```ts
import { createTestingPinia } from "@pinia/testing";

it("calls store action on click", async () => {
  const wrapper = mount(AddToCart, {
    props: { productId: 1 },
    global: {
      plugins: [
        createTestingPinia({
          stubActions: false,
        }),
      ],
    },
  });

  const cart = useCartStore();

  await wrapper.find("button").trigger("click");

  expect(cart.addItem).toHaveBeenCalledWith(1);
});
```

> 💡 `@pinia/testing` позволяет создавать мок-сторы. `stubActions: true` заменяет actions на `vi.fn()`, `stubActions: false` вызывает реальные actions.

---

## Тестирование с Vue Router

Для тестирования компонентов, зависящих от роутера, используйте `createRouter` с `createWebHistory` или `createMemoryHistory`.

```ts
import { createRouter, createWebHistory } from "vue-router";
import { routes } from "./router";

it("renders route component", async () => {
  const router = createRouter({
    history: createWebHistory(),
    routes,
  });

  router.push("/about");
  await router.isReady();

  const wrapper = mount(App, {
    global: {
      plugins: [router],
    },
  });

  expect(wrapper.text()).toContain("About us");
});
```

### Мокирование useRoute / useRouter

```ts
import { useRoute, useRouter } from "vue-router";

vi.mock("vue-router", async () => {
  const actual = await vi.importActual("vue-router");
  return {
    ...actual,
    useRoute: vi.fn(),
    useRouter: vi.fn(),
  };
});

it("reads query param", () => {
  useRoute.mockReturnValue({
    query: { q: "react" },
  });

  const wrapper = mount(SearchResults);

  expect(wrapper.text()).toContain("react");
});
```

---

## Тестирование provide / inject

`provide/inject` используется для передачи зависимость через дерево компонентов без пропс-дриллинга.

```ts
it("uses injected value", () => {
  const wrapper = mount(ThemedButton, {
    global: {
      provide: {
        theme: "dark",
      },
    },
  });

  expect(wrapper.find("button").classes()).toContain("btn--dark");
});
```

---

## Кастомные mount-обёртки

Если многие компоненты требуют одних и тех же плагинов (Pinia, router, i18n), создайте кастомную функцию `mount`.

```ts
// test-utils.ts
import { mount as vueMount } from "@vue/test-utils";
import { createPinia } from "pinia";
import { createRouter, createWebHistory } from "vue-router";
import { routes } from "../src/router";

export function mount(component, options = {}) {
  const pinia = createPinia();
  const router = createRouter({
    history: createWebHistory(),
    routes,
  });

  return vueMount(component, {
    global: {
      plugins: [pinia, router],
    },
    ...options,
  });
}
```

```ts
// MyComponent.test.ts
import { mount } from "../test-utils";
import MyComponent from "./MyComponent.vue";

it("renders", () => {
  const wrapper = mount(MyComponent);
  expect(wrapper.exists()).toBe(true);
});
```

---

## Лучшие практики

### 1. Тестируйте поведение, а не реализацию

```ts
// ❌ Хрупкий тест
expect(wrapper.vm.isOpen).toBe(true);

// ✅ Устойчивый тест
expect(wrapper.find(".modal").isVisible()).toBe(true);
```

### 2. Используйте data-testid как последний резерв

```ts
// ❌ Зависит от CSS-класса
wrapper.find(".btn-primary");

// ✅ Семантический поиск
wrapper.find("button");

// ✅ Когда нет другого способа
wrapper.find('[data-testid="submit"]');
```

### 3. Очищайте DOM и моки между тестами

```ts
// vitest.setup.ts
import { cleanup } from "@vue/test-utils";
import { afterEach, vi } from "vitest";

afterEach(() => {
  cleanup();
  vi.clearAllMocks();
});
```

### 4. Используйте setValue для форм

```ts
// ✅ Просто и надёжно
await wrapper.find("input").setValue("hello");

// ❌ Нужно вручную обновлять value и триггерить событие
wrapper.find("input").element.value = "hello";
await wrapper.find("input").trigger("input");
```

### 5. Один тест — одна проверка одного поведения

```ts
// ❌ Слишком много в одном тесте
it("handles form", async () => { ... });

// ✅ Разбито
it("shows error for empty email", async () => { ... });
it("emits submit with valid data", async () => { ... });
```

### 6. Мокируйте внешние зависимости через MSW

```ts
server.use(
  http.get("/api/users", () =>
    HttpResponse.json([{ id: 1, name: "Alice" }])
  )
);
```

---

## Антипаттерны

### 1. Тестирование внутреннего состояния

```ts
// ❌ Зависит от ref
expect(wrapper.vm.count).toBe(1);

// ✅ Проверяет UI
expect(wrapper.find("button").text()).toBe("Count: 1");
```

### 2. Использование querySelector и CSS-классов

```ts
// ❌ Хрупкий селектор
expect(wrapper.find("div > button.btn").exists()).toBe(true);

// ✅ Стабильный селектор
expect(wrapper.find("button").exists()).toBe(true);
```

### 3. Snapshot-тестирование всего компонента

```ts
// ❌ Ломается при любом изменении разметки
expect(wrapper.html()).toMatchSnapshot();

// ✅ Snapshot только для стабильных частей
expect(wrapper.find("svg").html()).toMatchSnapshot();
```

### 4. Зависимость тестов друг от друга

```ts
// ❌ Тесты делят состояние
const wrapper = mount(Counter);

it("increments", async () => {
  await wrapper.find("button").trigger("click");
  expect(wrapper.text()).toBe("Count: 1");
});

it("increments again", async () => {
  // Зависит от предыдущего теста
  await wrapper.find("button").trigger("click");
  expect(wrapper.text()).toBe("Count: 2");
});

// ✅ Каждый тест изолирован
it("increments from zero to one", async () => {
  const wrapper = mount(Counter);
  await wrapper.find("button").trigger("click");
  expect(wrapper.text()).toBe("Count: 1");
});
```

### 5. Игнорирование асинхронности

```ts
// ❌ Не дожидается обновления
it("loads data", () => {
  const wrapper = mount(UserList);
  expect(wrapper.text()).toContain("Alice");
});

// ✅ Дожидается завершения
it("loads data", async () => {
  const wrapper = mount(UserList);
  await flushPromises();
  expect(wrapper.text()).toContain("Alice");
});
```

### 6. Избыточное мокирование

```ts
// ❌ Мокирует Vue
vi.mock("vue", () => ({ ref: vi.fn() }));

// ✅ Мокирует только внешние зависимости
server.use(http.get("/api/data", () => HttpResponse.json({ data: [] })));
```

### 7. Тесты без assertions

```ts
// ❌ Тест всегда проходит
it("renders component", () => {
  mount(MyComponent);
});

// ✅ Есть проверка
it("renders component", () => {
  const wrapper = mount(MyComponent);
  expect(wrapper.text()).toContain("Hello");
});
```
