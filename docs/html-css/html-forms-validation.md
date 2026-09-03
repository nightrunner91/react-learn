# Формы, валидация и `ElementInternals`

Формы — это не просто набор полей ввода. Это контракт между пользователем, DOM и сервером: какие данные собирать, как их проверять и как сообщать об ошибках. В этой статье разбираем встроенную валидацию HTML, Constraint Validation API и то, как сделать собственный элемент полноценным участником формы через `ElementInternals`.

## Глубокий разбор

### Как форма собирает данные

Любой элемент формы, у которого есть атрибут `name`, участвует в отправке. Браузер собирает пары `name=value` и либо строит из них `application/x-www-form-urlencoded` тело, либо передаёт в `FormData`.

```html
<form id="profile">
  <input name="email" value="user@example.com">
  <input type="checkbox" name="newsletter" checked>
  <button type="submit">Отправить</button>
</form>
```

```js
const form = document.getElementById('profile');
const data = new FormData(form);

for (const [key, value] of data) {
  console.log(key, value); // email user@example.com, newsletter on
}
```

Важный нюанс: кнопка `<button type="submit">` внутри формы отправляет её, но если внутри формы несколько submit-кнопок, каждая может иметь собственные `name` и `value`, которые тоже попадают в данные при активации.

### Встроенная валидация HTML

Браузер умеет проверять поля до отправки без единой строчки JS:

- `required` — поле не должно быть пустым;
- `type="email"`, `type="url"`, `type="number"` — проверка формата;
- `min` / `max` — для чисел и дат;
- `minlength` / `maxlength` — для строк;
- `pattern` — регулярное выражение;
- `step` — для чисел и дат.

```html
<input
  type="email"
  name="email"
  required
  minlength="5"
  placeholder="user@example.com"
>
```

Если валидация не проходит, браузер блокирует отправку и показывает всплывающую подсказку с сообщением об ошибке. Это поведение можно переопределить через Constraint Validation API.

### Constraint Validation API

Каждый элемент формы реализует интерфейс `ConstraintValidation`. Его ключевые методы и свойства:

- `willValidate` — будет ли элемент валидироваться;
- `checkValidity()` — проверяет поле, не показывая UI;
- `reportValidity()` — проверяет поле и показывает нативную подсказку;
- `setCustomValidity(message)` — устанавливает кастомную ошибку;
- `validity` — объект `ValidityState` с флагами: `valueMissing`, `typeMismatch`, `patternMismatch`, `tooShort`, `tooLong`, `rangeUnderflow`, `rangeOverflow`, `stepMismatch`, `badInput`, `customError`, `valid`.

```js
const input = document.querySelector('input[name="password"]');

input.addEventListener('input', () => {
  if (input.value.length < 8) {
    input.setCustomValidity('Пароль должен быть не короче 8 символов');
  } else {
    input.setCustomValidity(''); // сброс кастомной ошибки
  }
});
```

Если `setCustomValidity('')` не вызвать, поле останется невалидным даже при корректном значении, потому что флаг `customError` будет установлен.

### Псевдоклассы валидации

Браузер применяет к элементам псевдоклассы:

- `:valid` / `:invalid` — поле проходит или не проходит валидацию;
- `:user-valid` / `:user-invalid` — то же самое, но только после взаимодействия пользователя с полем;
- `:required` / `:optional` — по наличию атрибута `required`;
- `:in-range` / `:out-of-range` — для числовых полей.

```css
input:user-invalid {
  border-color: #dc2626;
}

input:user-valid {
  border-color: #16a34a;
}
```

`:user-valid` и `:user-invalid` удобнее `:valid`/`:invalid`, потому что не подсвечивают поля красным сразу при загрузке страницы.

### Событие `submit` и `formdata`

При отправке формы происходит последовательность:

1. Событие `submit` на `<form>`.
2. Если обработчик не отменил событие и форма валидна, браузер инициирует навигацию или отправку.
3. Перед отправкой генерируется событие `formdata` на `<form>`, в обработчике которого можно модифицировать данные через `FormData`.

```js
form.addEventListener('submit', (event) => {
  event.preventDefault();
  const data = new FormData(form);
  fetch('/api/profile', { method: 'POST', body: data });
});

form.addEventListener('formdata', (event) => {
  event.formData.append('timezone', Intl.DateTimeFormat().resolvedOptions().timeZone);
});
```

### `ElementInternals`: собственный элемент внутри формы

`ElementInternals` — это API, которое позволяет custom element участвовать в жизни формы: валидироваться, сабмититься, реагировать на reset и disabled, восстанавливать состояние после навигации назад.

Чтобы связать custom element с формой:

1. Объявить статическое свойство `formAssociated = true`.
2. Получить `ElementInternals` через `this.attachInternals()`.
3. Устанавливать значение через `internals.setFormValue(value)`.

```js
class RatingElement extends HTMLElement {
  static formAssociated = true;

  constructor() {
    super();
    this._internals = this.attachInternals();
    this._value = '0';
  }

  connectedCallback() {
    this.addEventListener('click', this);
    this._internals.setFormValue(this._value);
  }

  handleEvent(event) {
    if (event.type === 'click') {
      this._value = event.target.dataset.value ?? this._value;
      this._internals.setFormValue(this._value);
    }
  }
}

customElements.define('star-rating', RatingElement);
```

```html
<form>
  <star-rating name="rating"></star-rating>
  <button type="submit">Отправить</button>
</form>
```

### Жизненный цикл form-associated элемента

Когда custom element участвует в форме, браузер вызывает дополнительные колбэки:

- `formAssociatedCallback(form)` — элемент ассоциирован с формой;
- `formDisabledCallback(disabled)` — состояние disabled формы изменилось;
- `formResetCallback()` — форма сброшена через `<button type="reset">` или `form.reset()`;
- `formStateRestoreCallback(state, mode)` — браузер восстанавливает состояние после навигации назад (`mode === 'restore'`) или autofill (`mode === 'autocomplete'`).

```js
class TokenInput extends HTMLElement {
  static formAssociated = true;

  constructor() {
    super();
    this._internals = this.attachInternals();
    this._value = '';
  }

  formResetCallback() {
    this._value = '';
    this._internals.setFormValue('');
    this.render();
  }

  formDisabledCallback(disabled) {
    this.toggleAttribute('disabled', disabled);
  }
}
```

### Валидация внутри custom element

`ElementInternals` позволяет устанавливать собственное состояние валидности:

```js
this._internals.setValidity(
  { valueMissing: true },
  'Выберите значение',
  this.firstElementChild
);
```

Первый аргумент — объект `ValidityStateFlags`, второй — сообщение об ошибке, третий — элемент, к которому привязать ошибку для фокуса. Чтобы пометить элемент валидным, передают пустой объект:

```js
this._internals.setValidity({});
```

После этого `this._internals.checkValidity()` и `this._internals.reportValidity()` работают так же, как у нативных элементов формы.

## Практические примеры

### Пример 1: валидация пароля

```html
<form id="signup">
  <label for="password">Пароль</label>
  <input
    id="password"
    name="password"
    type="password"
    minlength="8"
    pattern="^(?=.*[A-Za-z])(?=.*\d).+$"
    required
  >
  <span class="error" id="password-error"></span>

  <button type="submit">Зарегистрироваться</button>
</form>
```

```js
const form = document.getElementById('signup');
const password = form.elements.password;
const error = document.getElementById('password-error');

password.addEventListener('input', () => {
  password.setCustomValidity('');

  if (password.validity.valueMissing) {
    password.setCustomValidity('Введите пароль');
  } else if (password.validity.tooShort) {
    password.setCustomValidity(`Минимум ${password.minLength} символов`);
  } else if (password.validity.patternMismatch) {
    password.setCustomValidity('Пароль должен содержать буквы и цифры');
  }

  error.textContent = password.validationMessage;
});

form.addEventListener('submit', (event) => {
  if (!form.reportValidity()) {
    event.preventDefault();
  }
});
```

### Пример 2: кастомный слайдер рейтинга в форме

```js
class RatingInput extends HTMLElement {
  static formAssociated = true;

  constructor() {
    super();
    this._internals = this.attachInternals();
    this._value = '0';
  }

  connectedCallback() {
    this.render();
    this._internals.setFormValue(this._value);
  }

  render() {
    this.innerHTML = `
      <div class="rating" role="slider" aria-valuemin="1" aria-valuemax="5" aria-valuenow="${this._value}">
        ${[1, 2, 3, 4, 5].map(i => `<button type="button" data-value="${i}">${i}</button>`).join('')}
      </div>
    `;
  }

  handleEvent(event) {
    const button = event.target.closest('button');
    if (!button) return;

    this._value = button.dataset.value;
    this._internals.setFormValue(this._value);
    this.render();
  }

  formResetCallback() {
    this._value = '0';
    this._internals.setFormValue(this._value);
    this.render();
  }
}

customElements.define('rating-input', RatingInput);
```

```html
<form>
  <label for="review">Отзыв</label>
  <textarea id="review" name="review" required></textarea>

  <rating-input name="rating" required></rating-input>

  <button type="reset">Сбросить</button>
  <button type="submit">Отправить</button>
</form>
```

### Пример 3: модификация данных перед отправкой

```js
const form = document.getElementById('checkout');

form.addEventListener('formdata', (event) => {
  const data = event.formData;

  // нормализуем телефон
  const phone = data.get('phone').replace(/\D/g, '');
  data.set('phone', phone);

  // добавляем метаданные
  data.append('submittedAt', new Date().toISOString());
});

form.addEventListener('submit', async (event) => {
  event.preventDefault();

  if (!form.reportValidity()) return;

  await fetch('/api/order', {
    method: 'POST',
    body: new FormData(form)
  });
});
```

## Типичные ошибки и антипаттерны

- **Валидация только на сервере.** Клиентская валидация не заменяет серверную, но улучшает UX и уменьшает нагрузку. Делайте и то, и другое.
- **Сообщения об ошибках висят в HTML и не синхронизируются с `validationMessage`.** Дублируйте состояние из API, чтобы скринридеры и пользователи видели одно и то же.
- **Использование `:invalid` для стилизации сразу после загрузки.** При загрузке формы все обязательные пустые поля будут `:invalid`. Используй `:user-invalid` или классы после `reportValidity()`.
- **Забытый `setCustomValidity('')`.** Если один раз установить кастомную ошибку, она останется навсегда. Сбрасывайте её, когда условие исправлено.
- **Отмена `submit` без `event.preventDefault()` и без `reportValidity()`.** Проверяйте `form.reportValidity()` до отправки, иначе браузер может отправить невалидную форму.
- **Custom element без `formAssociated = true` пытается участвовать в форме.** Без этого свойства `attachInternals()` вернёт `internals`, но форма не увидит значение элемента.
- **Передача в `setFormValue` не строки.** Метод принимает `FormData`, `File` или строку. Если передать объект, получится непредсказуемое поведение.

## Ключевые тезисы для интервью

- Встроенная HTML-валидация работает через атрибуты `required`, `pattern`, `min`/`max`, `minlength`/`maxlength` и блокирует отправку формы при ошибках.
- Constraint Validation API даёт полный контроль: `checkValidity()`, `reportValidity()`, `setCustomValidity()` и объект `ValidityState`.
- `:user-valid` и `:user-invalid` удобнее `:valid`/`:invalid`, потому что не срабатывают до взаимодействия пользователя.
- `ElementInternals` позволяет custom element участвовать в форме: передавать значение через `setFormValue`, валидироваться, реагировать на `reset` и `disabled`.
- Form-associated custom element должен объявить `static formAssociated = true` и получить `this.attachInternals()`.
- Событие `formdata` позволяет модифицировать данные формы непосредственно перед отправкой.
- Клиентская валидация — это UX, а не безопасность. Сервер всегда должен проверять данные повторно.

## Полезные ссылки

- [Form validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation)
- [Constraint validation](https://developer.mozilla.org/en-US/docs/Web/HTML/Constraint_validation)
- [ElementInternals](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals)
- [Form-associated custom elements](https://web.dev/articles/more-capable-form-controls)
- [`formdata` event](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/formdata_event)
