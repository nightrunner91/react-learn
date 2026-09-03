# Custom elements, Shadow DOM и slots

Web Components — это набор стандартов, позволяющих создавать изолированные, переиспользуемые компоненты нативными браузерными средствами без фреймворка. В статье разбираем три кита этой экосистемы: custom elements для регистрации собственных тегов, Shadow DOM для инкапсуляции и slots для композиции контента.

## Глубокий разбор

### Custom elements: автономные и расширенные

Custom element — это класс, наследующий `HTMLElement`, который браузер регистрирует под определённым именем тега. Имена обязательно должны содержать дефис: `<my-component>`, `<user-card>`. Это защищает конфликт с будущими стандартными тегами.

```js
class GreeterElement extends HTMLElement {
  constructor() {
    super();
    this.textContent = 'Hello, Web Components!';
  }
}

customElements.define('my-greeter', GreeterElement);
```

```html
<my-greeter></my-greeter>
```

Существуют два типа custom elements:

- **Autonomous custom elements** — полностью новый тег, наследующий `HTMLElement`.
- **Customized built-in elements** — расширение существующего тега через `is="my-button"` и наследование от `HTMLButtonElement`. Поддержка в Safari ограничена, поэтому на практике почти не используются.

### Lifecycle callbacks

Браузер вызывает у custom element четыре основных колбэка:

- `connectedCallback()` — элемент добавлен в DOM. Здесь обычно рендерят разметку и вешают слушатели;
- `disconnectedCallback()` — элемент удалён из DOM. Место для отписки, остановки таймеров и очистки ресурсов;
- `adoptedCallback()` — элемент перемещён в другой документ, например через `adoptNode`;
- `attributeChangedCallback(name, oldValue, newValue)` — изменился один из наблюдаемых атрибутов. Список наблюдаемых атрибутов задаётся через статический getter `observedAttributes`.

```js
class CounterElement extends HTMLElement {
  static get observedAttributes() {
    return ['start'];
  }

  constructor() {
    super();
    this._count = 0;
  }

  connectedCallback() {
    this._count = Number(this.getAttribute('start') || 0);
    this.render();
    this.addEventListener('click', this);
  }

  disconnectedCallback() {
    this.removeEventListener('click', this);
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'start' && this.isConnected) {
      this._count = Number(newValue || 0);
      this.render();
    }
  }

  handleEvent() {
    this._count++;
    this.render();
  }

  render() {
    this.textContent = `Count: ${this._count}`;
  }
}

customElements.define('my-counter', CounterElement);
```

Важный нюанс: `attributeChangedCallback` не вызывается для атрибутов, которые не перечислены в `observedAttributes`. Также он не вызывается при первоначальной установке атрибута в `constructor`, потому что элемент ещё не зарегистрирован.

### Shadow DOM

Shadow DOM — это изолированное дерево внутри элемента. Оно отделяет внутреннюю разметку и стили от основного документа, предотвращая конфликты классов и просачивание стилей.

```js
class CardElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 16px;
          border: 1px solid #e5e7eb;
          border-radius: 8px;
        }
        .title { font-size: 1.25rem; margin: 0 0 8px; }
      </style>
      <h3 class="title"></h3>
      <div class="body"><slot></slot></div>
    `;

    this._title = shadow.querySelector('.title');
  }

  connectedCallback() {
    this._title.textContent = this.getAttribute('heading') || '';
  }
}

customElements.define('my-card', CardElement);
```

```html
<my-card heading="Заголовок карточки">
  <p>Контент карточки</p>
</my-card>
```

Режимы Shadow DOM:

- `open` — `element.shadowRoot` доступен извне;
- `closed` — `element.shadowRoot` возвращает `null`. Это не безопасность, а скорее соглашение: защиту можно обойти, если сохранить ссылку на shadow root внутри класса.

### Стилизация Shadow DOM

Стили внутри shadow root не влияют на внешний документ, и наоборот. Но есть механизмы для проброса стилей наружу и внутрь:

- `:host` — стилизует сам custom element;
- `:host(.theme-dark)` — применяется, если у элемента есть класс `theme-dark`;
- `:host-context(.theme-dark)` — применяется, если предок элемента имеет класс `theme-dark` (не поддерживается в Firefox);
- `::part(name)` — стилизует элемент внутри shadow root, помеченный атрибутом `part`;
- `::slotted(selector)` — стилизует элементы, распределённые в slot.

```css
my-card::part(icon) {
  color: #2563eb;
}
```

### Slots: композиция через теневое дерево

Slots позволяют вставлять светлый DOM (light DOM) внутрь shadow root. Браузер «распределяет» дочерние элементы компонента по слотам в теневом дереве.

- `<slot></slot>` — слот по умолчанию, принимает любые узлы без атрибута `slot`;
- `<slot name="header"></slot>` — именованный слот, принимает элементы с `slot="header"`;
- Внутри `<slot>` можно указать fallback-контент, который отобразится, если слот пуст.

```js
class LayoutElement extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' }).innerHTML = `
      <style>
        header { font-weight: bold; border-bottom: 1px solid #ccc; }
        footer { color: #666; font-size: 0.875rem; }
      </style>
      <header><slot name="header">Default header</slot></header>
      <main><slot>Default content</slot></main>
      <footer><slot name="footer"></slot></footer>
    `;
  }
}

customElements.define('my-layout', LayoutElement);
```

```html
<my-layout>
  <span slot="header">Шапка</span>
  <p>Основной контент</p>
  <span slot="footer">Подвал</span>
</my-layout>
```

### Событие `slotchange`

Когда в слот добавляются или удаляются распределённые узлы, слот генерирует событие `slotchange`. Его можно использовать, чтобы реагировать на изменение контента.

```js
const slot = this.shadowRoot.querySelector('slot');
slot.addEventListener('slotchange', () => {
  const nodes = slot.assignedNodes();
  console.log('Распределено узлов:', nodes.length);
});
```

`assignedNodes()` возвращает все узлы, `assignedElements()` — только элементы. Оба метода не возвращают fallback-контент.

### События и composed

События, всплывающие из Shadow DOM, подвергаются ретаргетингу (retargeting): `event.target` становится самим custom element, а не внутренним узлом. Это защищает инкапсуляцию, но иногда мешает понять, что именно кликнул пользователь.

Чтобы событие пробило границу Shadow DOM, у него должен быть флаг `composed: true`. Встроенные события вроде `click` имеют этот флаг, а `focus`/`blur` — нет. Кастомные события нужно создавать с `composed: true` явно:

```js
this.dispatchEvent(new CustomEvent('item-select', {
  bubbles: true,
  composed: true,
  detail: { id: 42 }
}));
```

### Declarative Shadow DOM

Declarative Shadow DOM — это способ объявить shadow root прямо в HTML без JS:

```html
<my-card>
  <template shadowrootmode="open">
    <style>:host { display: block; }</style>
    <slot></slot>
  </template>
  <p>Содержимое карточки</p>
</my-card>
```

Это полезно для SSR и первоначального рендера, но поддержка пока не везде одинаковая.

## Практические примеры

### Пример 1: пользовательская карточка

```js
class UserCard extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        :host {
          display: flex;
          gap: 12px;
          align-items: center;
          padding: 12px;
          border: 1px solid #e5e7eb;
          border-radius: 8px;
          font-family: system-ui, sans-serif;
        }
        ::slotted(img) {
          width: 48px;
          height: 48px;
          border-radius: 50%;
        }
        .name { font-weight: 600; }
        .role { color: #6b7280; font-size: 0.875rem; }
      </style>
      <slot name="avatar"></slot>
      <div>
        <div class="name"><slot name="name"></slot></div>
        <div class="role"><slot name="role">Unknown role</slot></div>
      </div>
    `;
  }
}

customElements.define('user-card', UserCard);
```

```html
<user-card>
  <img slot="avatar" src="avatar.png" alt="">
  <span slot="name">Анна Смирнова</span>
  <span slot="role">Frontend-разработчик</span>
</user-card>
```

### Пример 2: вкладки с Shadow DOM

```js
class TabPanel extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' }).innerHTML = `
      <style>
        :host { display: block; }
        ::slotted([slot="panel"]) { display: none; }
        ::slotted([slot="panel"][active]) { display: block; }
      </style>
      <div class="tabs"><slot name="tab"></slot></div>
      <slot name="panel"></slot>
    `;
  }

  connectedCallback() {
    this.addEventListener('click', (event) => {
      const tab = event.target.closest('[slot="tab"]');
      if (!tab) return;

      const index = Array.from(this.querySelectorAll('[slot="tab"]')).indexOf(tab);
      this.selectTab(index);
    });
  }

  selectTab(index) {
    const tabs = this.querySelectorAll('[slot="tab"]');
    const panels = this.querySelectorAll('[slot="panel"]');

    tabs.forEach((t, i) => t.toggleAttribute('active', i === index));
    panels.forEach((p, i) => p.toggleAttribute('active', i === index));
  }
}

customElements.define('tab-panel', TabPanel);
```

```html
<tab-panel>
  <button slot="tab" active>Вкладка 1</button>
  <button slot="tab">Вкладка 2</button>

  <div slot="panel" active>Содержимое 1</div>
  <div slot="panel">Содержимое 2</div>
</tab-panel>
```

### Пример 3: lifecycle и атрибуты

```js
class TooltipElement extends HTMLElement {
  static get observedAttributes() {
    return ['text', 'position'];
  }

  constructor() {
    super();
    this.attachShadow({ mode: 'open' }).innerHTML = `
      <style>
        :host { position: relative; display: inline-block; }
        .tip {
          position: absolute;
          padding: 4px 8px;
          background: #1f2937;
          color: #fff;
          border-radius: 4px;
          font-size: 0.75rem;
          white-space: nowrap;
          opacity: 0;
          transition: opacity 0.2s;
          pointer-events: none;
        }
        :host(:hover) .tip { opacity: 1; }
      </style>
      <slot></slot>
      <span class="tip"></span>
    `;
    this._tip = this.shadowRoot.querySelector('.tip');
  }

  connectedCallback() {
    this.update();
  }

  attributeChangedCallback() {
    this.update();
  }

  update() {
    this._tip.textContent = this.getAttribute('text') || '';
    this._tip.className = `tip ${this.getAttribute('position') || 'top'}`;
  }
}

customElements.define('my-tooltip', TooltipElement);
```

```html
<my-tooltip text="Подсказка" position="bottom">
  <button>Наведи на меня</button>
</my-tooltip>
```

## Типичные ошибки и антипаттерны

- **Работа с DOM в `constructor`.** В `constructor` элемент ещё не в документе, поэтому `this.querySelector` ищет только внутри самого элемента, а атрибуты могут быть ещё не установлены. Тяжёлый рендер и обращение к потомкам лучше откладывать в `connectedCallback`.
- **Отсутствие очистки в `disconnectedCallback`.** Таймеры, подписки, глобальные слушатели остаются висеть и вызывают утечки памяти.
- **Забытый `super()` в `constructor`.** Custom element обязан вызвать `super()` до любого обращения к `this`, иначе будет ошибка.
- **Использование `closed` shadow root ради безопасности.** `closed` не скрывает реализацию от опытного разработчика, он лишь усложняет отладку и тестирование. Используйте `open` по умолчанию.
- **Попытка стилизовать внутренности компонента обычными селекторами.** Стили основного документа не проникают в Shadow DOM. Используйте CSS custom properties, `::part` или специально пробрасываемые переменные.
- **Расчёт на `attributeChangedCallback` для любых атрибутов.** К回调 срабатывает только для атрибутов из `observedAttributes`.
- **Отсутствие `composed: true` у кастомных событий.** Если событие должно быть услышано снаружи компонента, создавайте его с `composed: true`.
- **Передача сложных данных через атрибуты.** Атрибуты — строки. Для объектов и массивов используйте свойства класса, а не `setAttribute`.

## Ключевые тезисы для интервью

- Custom element регистрируется через `customElements.define('my-tag', MyClass)`; имя обязательно содержит дефис.
- Основные lifecycle callbacks: `connectedCallback`, `disconnectedCallback`, `adoptedCallback`, `attributeChangedCallback` + статический `observedAttributes`.
- Shadow DOM создаёт изолированное дерево через `attachShadow({ mode: 'open' | 'closed' })`; стили и разметка внутри не просачиваются наружу.
- Slots распределяют light DOM внутрь shadow root: `<slot>` — по умолчанию, `<slot name="x">` — именованный слот.
- События из Shadow DOM подвергаются retargeting; кастомные события, которые должны всплыть наружу, нужно создавать с `composed: true`.
- `:host`, `::part`, `::slotted` и CSS custom properties — основные способы стилизовать компоненты снаружи и изнутри.
- `closed` shadow root не даёт реальной безопасности, а лишь затрудняет отладку; предпочитайте `open`.

## Полезные ссылки

- [Using custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)
- [Using shadow DOM](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM)
- [Using templates and slots](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_templates_and_slots)
- [Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components)
- [Custom Elements Everywhere](https://custom-elements-everywhere.com/)
