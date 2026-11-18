# ATLAS UI · CSS Architecture Rescue

Учебный стенд ARCH-06/07: компонентная CSS-библиотека из 7 семейств, две темы, vendor-виджет подрядчика.
**Исправленная версия** (ветка `fix/css-architecture`): архитектура `@layer` + семантические токены, 18/18 структурных тестов, 6/6 браузерных сценариев.

## Запуск

1. Установите Node.js 18 или новее.
2. В терминале из корня проекта: `npm run dev` (никаких зависимостей для запуска не нужно).
3. Откройте `http://127.0.0.1:5173` в современном браузере.
4. `npm test` — 18 структурных проверок (после рефакторинга: 18/18; baseline-коммит `848810a`: 4/18).
5. Не открывайте HTML через `file://`: используйте локальный сервер.

## Браузерные тесты

```bash
npm install --save-dev @playwright/test
npx playwright install chromium
npm run test:e2e
```

Шесть сценариев: модальность диалога и возврат фокуса, клавиатурный доступ к переключателю, отсутствие ошибки до взаимодействия, отсутствие горизонтального overflow на 6 ширинах, смена темы для primary-кнопки, закрытие/восстановление уведомлений. После рефакторинга: 6/6 (baseline: 1/6).

## Архитектура (итог рефакторинга)

Порядок слоёв в `css/app.css` (контракт — см. `reports/ADR_TEMPLATE.md`):

```
@layer reset, legacy, tokens, base, components, states, utilities;
@import url('./legacy.css') layer(legacy);   /* vendor изолирован ранним слоем */
```

- **vendor CSS** сохранён, но ограничен областью `.legacy-widget` и импортирован в ранний слой `legacy`;
- **все цвета компонентов** — семантические токены (`--color-accent`, `--color-danger`, …) из `css/tokens.css`; смена темы = смена значений переменных;
- **состояния** (`hover`, `focus-visible`, `active`, `disabled`, `checked`, ошибка поля) живут в позднем слое `states`; ошибка поля управляется атрибутом `aria-invalid` (JS после blur/submit), а не преждевременным `:invalid`;
- **переключатель** — нативный `input[type=checkbox]`, визуально скрытый (не `display:none`), focus-visible обводка на треке;
- **диалог** открывается `showModal()`: top layer, inert-фон, Escape, возврат фокуса;
- глобальные служебные классы (`.is-danger`), ID-селекторы и `!important` в библиотеке отсутствуют.

## Отчётные материалы

- `reports/AUDIT_TEMPLATE.md` — аудит: 18 первопричин (A-01…A-18) с трассировками каскада;
- `reports/ADR_TEMPLATE.md` — ADR-001: сравнение 4 стратегий, контракт слоёв и публичный API семейств;
- `reports/MATRIX_TEMPLATE.csv` — 64 сценария (до/после, 33 исправлены с FAIL на OK);
- `reports/REGRESSION_TEMPLATE.md` — регрессионный отчёт с результатами прогонов;
- `reports/screenshots/before|after/` — скриншоты 6 ширин × 2 темы, диалог, фокус, поле.

## История коммитов

1. `baseline: inherited CSS conflicts` — исходная заготовка с намеренными дефектами (main);
2. `audit: 18 root causes + 3 cascade traces (AUDIT, ADR)` — расследование и архитектурное решение;
3. `fix: cascade layers architecture, 7 component families, states` — реализация;
4. `test: all suites green + report artifacts` — финальные прогоны и артефакты.

## Ограничения (честно)

Контраст disabled-элементов оценивается до наложения opacity; hover-состояния не тестируются на touch-ширинах; попиксельный visual-regression не автоматизирован. Подробности и планы — в `reports/REGRESSION_TEMPLATE.md`.
