# 01. Аудит CSS-архитектуры

Автор / группа: студент группы ИСИТ-4 · Репозиторий: atlas-ui (ветка `fix/css-architecture`) · Commit до правок: `848810a baseline: inherited CSS conflicts`

Исходное состояние (замер `npm test` / `npm run test:e2e` / скрипт matrix.mjs на baseline):

- `npm test` — **4 / 18** пройдено (провалены 02, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 17, 18);
- `npm run test:e2e` — **1 / 6** пройдено (провалены: модальность диалога, клавиатурный доступ к переключателю, валидация после blur, overflow на 320/375, смена темы для primary-кнопки);
- матрица из 64 программных сценариев — **31 OK / 33 FAIL** (`reports/MATRIX_TEMPLATE.csv`, колонка «Факт_до»).

## Гипотеза о причине (до изменения кода)

После интеграции `legacy.css` стили подрядчика попали в документ как **unlayered-правила**: при normal-важности они сильнее **любого** именованного слоя независимо от специфичности селекторов (`@layer` сравнивается до специфичности, а unlayered-блоки стоят выше всех именованных слоёв). Поэтому точечные правила подрядчика (`#workbench .ui-button`, `.ui-card p`, `button { border-radius: 0 }`, `input { outline: none }`) перебивают всю библиотеку Atlas UI. `@layer` это решает, если импортировать vendor-файл в ранний слой `layer(legacy)`. Но `@layer` **не** решает: (а) `!important` внутри vendor-файла — для important порядок слоёв обратный, ранний слой становится сильнее; (б) семантические дефекты разметки и JS (`display:none` у чекбокса, `dialog.show()` вместо `showModal()`, преждевременный `:invalid`); (в) утечку глобального служебного класса `.is-danger`, который сам по себе не связан с vendor-файлом и живёт в нашем же `states.css`.

## Инвентаризация дефектов (минимум 12 доказанных причин)

Доказательства — computed styles из DevTools-эквивалента (скрипты `scripts/probe.mjs`, `scripts/matrix.mjs` на baseline-коммите) и скриншоты `reports/screenshots/before/`.

| ID | Симптом и шаги воспроизведения | Файл:строка | Применённое правило и выигравший каскад | Первопричина | Минимальное исправление | Регрессионный тест |
|---|---|---|---|---|---|---|
| A-01 | Primary-кнопка в стенде тёмно-синяя, не меняется при смене темы: `background-color: rgb(41, 57, 168)` вместо `--color-accent` | `css/legacy.css:3` | `#workbench .ui-button { background:#2939a8 }` — unlayered author normal, специфичность 1-1-0. Побеждает `@layer components .ui-button--primary` (0-1-0): unlayered-блок выше любого именованного слоя при normal | Vendor CSS импортирован без `layer()` | `@import url('./legacy.css') layer(legacy)` + удалить нацеливание на библиотеку из vendor-файла | Тест 02, 18; e2e «primary button uses theme tokens»; сценарии #1/#4 матрицы |
| A-02 | Все кнопки «квадратные»: `border-radius: 0` (в т.ч. `.ui-button`) | `css/legacy.css:2` | `button { border-radius: 0 }` — unlayered, 0-0-1, но слой важнее специфичности → побеждает `@layer components .ui-button { border-radius: var(--radius-md) }` (0-1-0) | Глобальный селектор тега в vendor-файле | Ограничить vendor-стиль его областью: `.legacy-widget button { … }` внутри `layer(legacy)` | Сценарий #64 матрицы (legacyRadius=0px, uiRadius=12px — различаются) |
| A-03 | Опасная кнопка «Удалить» игнорирует варианты: `#a41e24 !important` | `css/legacy.css:4` | `#workbench .ui-button.is-danger { background:#a41e24 !important }` — author **important**, 1-2-0. Important сравнивается до normal-объявлений всей библиотеки | `!important` + ID-селектор в vendor CSS | Удалить правило (оно нацелено на нашу библиотеку); вариант-цвет задаёт `--danger`-токен | Тест 03 (нет `!important`), 18 |
| A-04 | Метка «Риск» в карточке №03 выглядит как кнопка удаления (ярко-красная, белый текст): `bg rgb(193,34,54)`, `color #fff` | `css/states.css:11` | `.is-danger { background:#c12236; color:#fff }` — `@layer states`, 0-1-0, стоит **позже** в порядке слоёв, чем `components` → для normal поздний слой сильнее, побеждает над `.ui-tag--danger` (слой components) | Глобальный служебный класс состояния применён к разным семействам (тег, alert, кнопка) | Удалить `.is-danger` из CSS и разметки; каждое семейство несёт свой модификатор `--danger` | Тесты 08, 17; сценарии #26, #38 матрицы |
| A-05 | Поле «Корпоративная почта» красное ещё до ввода: `border-color: rgb(180,35,54)`, `bg rgb(255,240,242)` | `css/states.css:9-10` | `.ui-field:has(input:invalid) input` — `:invalid` у `required`-поля наступает сразу в разметке (пустое значение), до всякого взаимодействия | Семантическая ошибка: CSS-псевдокласс `:invalid` использован как UX-состояние «ошибка после взаимодействия» | Перейти на `input[aria-invalid='true']`, которым управляет JS после blur/submit | Тест 09; e2e «field does not display error before before user interacts»; сценарии #14-#15 |
| A-06 | Переключатель недоступен с клавиатуры: Tab пропускает, Space не работает, `display: none` | `css/components.css:44` | `.ui-switch input { display: none }` — элемент исключён из дерева доступности и порядка табуляции | Визуальное скрытие нативного контрола вместо паттерна visually-hidden | `position:absolute; width/height:1px; opacity:0` + `:focus-visible + .ui-switch__track { outline }` | Тесты 05, 06; e2e «switch is keyboard reachable»; сценарии #39-#41 |
| A-07 | «Модальное» окно не модально: фон прокручивается, Tab уходит под окно, `:modal = false` | `js/app.js:15` | `dialog.show()` — открывает **не**модальный диалог (без top layer, inert-фона, ловушки фокуса, возврата фокуса) | Неправильный API открытия | `dialog.showModal()` | Тест 04; e2e «native dialog is truly modal»; сценарии #45-#46 |
| A-08 | Горизонтальный скролл на 320/375 px: `scrollWidth 380 > 320` | `css/components.css:12,17` | `.ui-field input { min-width: 20rem }`, `.ui-card { min-width: 22rem }` — фиксированные min-width шире мобильного трека grid | min-width ребёнка больше трека сетки | `min-width: 0` на флекс/grid-детей + `width: 100%` вместо min-width | Тест 10; e2e overflow-сценарий; сценарии #21-#22, #49-#52 |
| A-09 | Текст во вложенном уведомлении карликовый: `.ui-alert p { font-size: 11.04px }` | `css/legacy.css:6` | `.ui-card p { font-size:.69rem }` — unlayered 0-1-1. Бьёт и по `p` внутри чужого компонента `.ui-alert`, вложенного в карточку | Селектор потомка без ограничения владения: карточка оформляет **всех** потомков, а не свои слоты | В vendor-файле ограничить областью `.legacy-widget`; в библиотеке — прямые селекторы `.ui-card__description` / `.ui-card > p` | Сценарии #23-#24 матрицы |
| A-10 | Псевдо-интерактивность меток: `cursor: pointer`, `filter: brightness(.94)` на hover у неинтерактивного тега | `css/states.css:13` | `.ui-tag:hover` — `@layer states` 0-2-0. Модифицирует неинтерактивный элемент | Состояние hover без действия = ложный афорданс | Удалить hover-правило тега | Сценарий #26 матрицы (cursor=auto/default) |
| A-11 | Разный padding тегов: `:is(.ui-card, #workbench) .ui-tag { padding-inline: 1rem }` даёт 16px везде вместо базовых 10.4px | `css/states.css:12` | `:is(.ui-card, #workbench) .ui-tag` — 0-2-0; вес `:is()` равен максимуму аргументов; правило меняет вид чужого компонента в зависимости от контекста | Контекстно-зависимая стилизация компонента от родителя | Удалить правило: тег един во всех контекстах | Тест 13; сценарий #26 (padCard = padTags = базовый) |
| A-12 | Фокус кнопок невидим: `outline: none` на `.ui-button:focus` и на всех `input` | `css/states.css:6`, `css/legacy.css:7` | `.ui-button:focus { outline:none }` (states, 0-2-0) и `input { outline:none }` (unlayered 0-0-1 → выше слоя states для normal) | Удаление обводки без замены; vendor-правило бьёт по `input:focus` даже из слоя | `:focus-visible` с обводкой из `--color-focus`; vendor ограничить `.legacy-widget` | Тест 05; сценарии #3, #20 |
| A-13 | Недоступная (disabled) кнопка показывает `cursor: pointer`: `{"cursor":"pointer","opacity":"0.45"}` | `css/states.css:8` | `.ui-button:disabled { opacity:.45; pointer-events:none }` — pointer-events отключает клики, но не сообщает пользователю о недоступности | Нативное состояние не завершено UX-подтверждением | Добавить `cursor: not-allowed` | Сценарии #8-#9 матрицы |
| A-14 | Опасная кнопка не реагирует на hover: `bg rgb(164,30,36)` = `#a41e24 !important` из A-03 перебивает `--danger-hover` | `css/states.css:5` vs `css/legacy.css:4` | important author origin выше всех normal-объявлений (unlayered vs layered не имеет значения) | Следствие A-03 | Устраняется вместе с A-03 | Сценарий #7 матрицы |
| A-15 | Дублирование базового блока кнопки: два правила `.ui-button { … }` подряд (второе правит letter-spacing и padding-inline) | `css/components.css:2-3` | Два блока 0-1-0 в одном слое — побеждает поздний по порядку; дубль рассеивает контракт компонента | Базовый селектор объявлен дважды | Слить в один блок | Тест 11 |
| A-16 | Жёсткие hex-цвета в компонентах: `#3659d9`, `#b42336`, `#971d2a`, `#111827` — тёмная тема их не меняет | `css/components.css:4,7,29` | `.ui-button--primary { background:#3659d9 }` — литерал вместо `var(--color-accent)` | Прямое значение вместо семантического токена | Перевести все варианты на токены | Тест 07; e2e theme-тест |
| A-17 | Нет reduced-motion: `transition 0.16s` остаётся при `prefers-reduced-motion: reduce` | `css/*` | Правила переходов не обёрнуты в медиазапрос | Не учтена настройка доступности | `@media (prefers-reduced-motion: reduce) { … transition-duration: 0.01ms }` | Тест 12; сценарий #63 |
| A-18 | Трек переключателя `#adb7c6` и alert-текст `#111827` — литералы, в dark не адаптируются | `css/components.css:45,29` | Литеральные цвета вне системы токенов | Частный случай A-16 в состояниях checked/уведомлений | Токены `--color-switch-track`, наследование `--color-ink` | Тест 07; сценарии #34-#37 |

## Три трассировки каскада

Формат: **элемент → CSS-свойство → кандидаты (origin / важность / layer / специфичность / порядок) → победившее правило и причина**.

### 1. Primary-кнопка в `#workbench` → `background-color`

Кандидаты (до исправления):

1. `@layer components { .ui-button--primary { background:#3659d9 } }` — author normal, слой components, 0-1-0;
2. `@layer states { .ui-button--primary:hover { … } }` — только для hover;
3. **`#workbench .ui-button { background:#2939a8 }`** (legacy.css, `@import` без `layer()`) — author normal, **unlayered**, 1-1-0;
4. `@layer tokens` — background не задаёт.

Победило **№3**: оба объявления author normal, но в Cascade Level 5 unlayered-стили стоят **выше** любых именованных слоёв; слой сравнивается раньше специфичности, поэтому даже ID здесь не потребовался бы. Доказательство: `rgb(41, 57, 168)` = `#2939a8` (computed на baseline; скриншот `before_1440_light.png`, probe-строка «btn-primary»). После исправления (импорт `layer(legacy)`) побеждает №1 — поздний слой components сильнее раннего legacy при normal, тема снова управляет кнопкой через `var(--color-accent)`.

### 2. Метка `.ui-tag--danger.is-danger` внутри карточки №03 → `background-color`

Кандидаты (до исправления):

1. `@layer components { .ui-tag--danger { background: var(--color-danger-bg); color: var(--color-danger) } }` — 0-1-0;
2. **`@layer states { .is-danger { background:#c12236; color:#fff } }`** — 0-1-0;
3. unlayered `.is-danger { color:#fff }` (legacy.css) — 0-1-0;
4. `@layer legacy` (после изоляции) — тегов не касается.

Победило **№2** (а до изоляции legacy — №2 вместе с №3): правила №1 и №2 оба 0-1-0, оба author normal, но **слой states объявлен позже components** — у normal-объявлений более поздний слой сильнее при равной специфичности. Корень проблемы — не специфичность, а то, что один глобальный класс состояния служит API сразу для трёх семейств (тег, alert, кнопка). Исправление: удалить `.is-danger` целиком; тег снова берёт `--color-danger-bg`/`--color-danger`. Доказательство: probe-строка «tag-danger в карточке №03»: `rgb(193, 34, 54)` до → токенный фон после.

### 3. `input#user-email [required]` до взаимодействия → `border-color`

Кандидаты (до исправления):

1. `@layer components { .ui-field input { border: 1px solid var(--color-border) } }` — 0-1-1;
2. **`@layer states { .ui-field:has(input:invalid) input { border-color:#b42336; background:#fff0f2 } }`** — 0-3-1 (аргумент `:has()` входит в вес);
3. unlayered `input { outline: none }` (legacy) — не про border;
4. JS устанавливает `aria-invalid` только после blur/submit — до этого атрибута нет.

Победило **№2**: псевдокласс `:invalid` у пустого `required`-поля истинен **сразу после парсинга HTML** (проверяется значение, а не взаимодействие), поэтому правило применялось до всякого касания поля. Это не дефект специфичности — усиление селектора №1 только ухудшило бы каскад. Исправление — сменить **источник состояния**: `.ui-field:has(input[aria-invalid='true'])`, где атрибут ставит JS по факту blur/submit (`js/app.js:20`). Тогда до взаимодействия побеждает №1 (`rgb(213,219,234)`), после blur на пустом поле — №2.

## Вывод

Проблема архитектуры (решается слоями и контрактом): unlayered-приоритет vendor CSS (A-01, A-02, A-09, часть A-12), глобальный класс состояния `.is-danger` (A-04), контекстные селекторы `:is()` (A-11), селекторы потомков (A-09), дубль базового блока (A-15), литеральные цвета вместо токенов (A-16, A-18), отсутствие reduced-motion (A-17), фиксированные min-width (A-08), неполный disabled (A-13). Поведение JS/семантики (слоями не решается): `dialog.show()` вместо `showModal()` (A-07), `display:none` у чекбокса (A-06), преждевременный `:invalid` (A-05), ложный hover у тега (A-10 — правило живёт в CSS, но это ошибка контракта «состояние — только у интерактивных», исправлена удалением правила). Ключевой урок: `@layer` управляет **приоритетом между группами правил**, но не заменяет семантику HTML, JS-логику состояний и дисциплину именования.
