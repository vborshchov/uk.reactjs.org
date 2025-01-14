---
id: hooks-rules
title: Правила хуків
permalink: docs/hooks-rules.html
next: hooks-custom.html
prev: hooks-effect.html
---

*Хуки* — це нововведення в React 16.8. Вони дозволяють вам використовувати стан та інші можливості React без написання класів.

Хуки — звичайні JavaScript функції, але ви маєте дотримуватися двох правил, використовуючи їх. Ми зробили [плагін для лінтеру](https://www.npmjs.com/package/eslint-plugin-react-hooks), щоб автоматично дотримуватися цих правил:

### Використовуйте хуки тільки на вищому рівні {#only-call-hooks-at-the-top-level}

**Не використовуйте хуки усередині циклів, умовних операторів або вкладених функцій.** Замість цього завжди використовуйте хуки на вищому рівні React-функцій. Дотримуючись цього правила, ви будете певні, що хуки викликаються в однаковій послідовності кожного разу, коли рендериться компонент. Це дозволяє React коректно зберігати стан хуків між численними викликами `useState` та `useEffect`. (Якщо вам цікаво, то ми пояснимо це більш детально [нижче](#explanation).)

### Викликайте хуки лише з React-функцій {#only-call-hooks-from-react-functions}

**Не викликайте хуки зі звичайних JavaScript-функцій.** Натомість, ви можете:

* ✅ Викликати хуки з функціонального компоненту React.
* ✅ Викликати хуки з користувацьких хуків (ми навчимося це робити [на наступній сторінці](/docs/hooks-custom.html)).

Дотримуючись цього правила, ви можете бути певні, що вся логіка компоненту зі станом чітко проглядається в його вихідному коді.

## Плагін для ESLint {#eslint-plugin}

Ми випустили плагін для ESLint [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks), який примушує дотримуватися цих двох правил. Ви можете додати цей плагін до вашого проекту, якщо ви хочете його спробувати:

```bash
npm install eslint-plugin-react-hooks --save-dev
```

```js
// Ваша конфігурація ESLint
{
  "plugins": [
    // ...
    "react-hooks"
  ],
  "rules": {
    // ...
    "react-hooks/rules-of-hooks": "error", // Перевіряє правила хуків
    "react-hooks/exhaustive-deps": "warn" // Перевіряє ефект залежностей
  }
}
```

Цей плагін включений за замовчуванням до [Create React App](/docs/create-a-new-react-app.html#create-react-app).

**Ви можете пропустити залишок сторінки та перейти до наступної, яка пояснює як писати [користувацькі хуки](/docs/hooks-custom.html).** На цій сторінці ми продовжимо і надамо пояснення необхідності цих правил.

## Пояснення {#explanation}

Як ми [дізналися раніше](/docs/hooks-state.html#tip-using-multiple-state-variables), в одному компоненті можна багаторазово використовувати хуки стану або ефектів:

```js
function Form() {
  // 1. Використовуємо змінну стану name
  const [name, setName] = useState('Ліна');

  // 2. Використовуємо ефект для збереження стану форми
  useEffect(function persistForm() {
    localStorage.setItem('formData', name);
  });

  // 3. Використовуємо змінну стану state
  const [surname, setSurname] = useState('Костенко');

  // 4. Використовуємо ефект, щоб оновити заголовок сторінки
  useEffect(function updateTitle() {
    document.title = name + ' ' + surname;
  });

  // ...
}
```

Отже, як React дізнається який стан відповідає певному виклику `useState`? Відповідь наступна: **React покладається на послідовність викликів хуків**. Наш приклад працює тому, що послідовність викликів хуків є сталою для кожного рендеру:

```js
// ------------
// Перший рендер
// ------------
useState('Ліна')           // 1. Ініціюємо змінну name зі значенням 'Ліна'
useEffect(persistForm)     // 2. Додаємо ефект, щоб зберегти данні форми
useState('Костенко')        // 3. Ініціюємо змінну surname зі значенням 'Костенко'
useEffect(updateTitle)     // 4. Додаємо ефект, щоб оновити заголовок сторінки

// -------------
// Другий рендер
// -------------
useState('Ліна')           // 1. Зчитуємо змінну стану name (аргумент ігнорується)
useEffect(persistForm)     // 2. Змінюємо ефект, щоб зберегти данні форми
useState('Костенко')        // 3. Зчитуємо змінну стану surname (аргумент ігнорується)
useEffect(updateTitle)     // 4. Змінюємо ефект, щоб оновити заголовок сторінки

// ...
```

Доки послідовність викликів хуків залишається сталою між рендерами, React може співвідносити локальний стан між кожним з них. Але, що трапиться, якщо ми розмістимо виклик хуку (наприклад, ефект `persistForm`) всередину умовного оператору?

```js
  // 🔴 Ми порушуємо перше правило, розміщуючи хук всередині умовного оператору
  if (name !== '') {
    useEffect(function persistForm() {
      localStorage.setItem('formData', name);
    });
  }
```

Умова `name !== ''` дорівнює `true` при першому рендері, тому цей хук буде виконано. Хай там що, та в наступному рендері користувач може очистити форму і таким чином змінити цю умову на `false`. Тепер, оскільки ми пропускаємо цей хук під час рендеру, послідовність викликів хуків стає іншою:

```js
useState('Ліна')           // 1. Зчитуємо змінну стану name (аргумент ігнорується)
// useEffect(persistForm)  // 🔴 Цей хук пропущено!
useState('Костенко')        // 🔴 2 (але був 3). Помилка при зчитуванні змінної стану surname
useEffect(updateTitle)     // 🔴 3 (but was 4). Помилка при зміні ефекту
```

React не знатиме, що повернути для другого виклику хуку `useState`. React очікував, що другий виклик хуку в цьому компоненті відповідає ефекту `persistForm` так само як і під час попереднього рендеру, але це більше не так. З цього моменту кожен наступний виклик хуку після того, що ми пропустили, також зміститься на один, що призведе до помилок.

**Ось чому хуки мають викликатися на вищому рівні наших компонентів.** Якщо ми хочемо викликати ефект за певної умови, то ми можемо розмістити цю умову *всередину* нашого хуку:

```js
  useEffect(function persistForm() {
    // 👍 Більше ми не порушимо перше правило
    if (name !== '') {
      localStorage.setItem('formData', name);
    }
  });
```

**Зауважте, що вам не потрібно буде піклуватися про цю проблему, якщо ви додасте до вашого проекту [запропоноване правило для лінтера](https://www.npmjs.com/package/eslint-plugin-react-hooks).** Але тепер ви знаєте, *чому* хуки працюють таким чином, та які проблеми можна запобігти використовуючи це правило.

## Наступні кроки {#next-steps}

Нарешті ми можемо почати вчитися тому, [як писати користувацькі хуки](/docs/hooks-custom.html)! Хуки користувача дозволять вам комбінувати хуки впроваджені React з вашими власними абстракціями та повторно використовувати загальну логіку стану між різними компонентами.
