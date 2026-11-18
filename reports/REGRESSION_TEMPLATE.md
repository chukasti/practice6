# 04. Регрессионный отчёт

## Сборка и окружение

Node: v24.22.3 · Browser: Chromium 159.0.8154.6 (Playwright headless) · Commit: baseline `848810a` → fix `fix/css-architecture` (см. git log) · ОС: Linux x64

## Тестирование

- `npm test`: **18 / 18** пройдено (baseline: 4/18 — провалены 02, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 17, 18).
- `npm run test:e2e`: **6 / 6** пройдено (baseline: 1/6).
- 320 / 375 / 768 / 1024 / 1440 / 1920 px: **нет горизонтального скролла ни на одной ширине** (e2e-тест «no horizontal viewport overflow…» + 12 строк матрицы #49–#60; baseline: overflow на 320 и 375 в обеих темах, scrollWidth 380 > 320).
- light / dark / reduced-motion / Tab / Shift+Tab / Space / Enter / Escape: **все проверки OK** (матрица #3, #6, #20, #39–#46, #61–#63; reduced-motion: transition-duration → 0.01ms при эмуляции `prefers-reduced-motion: reduce`).
- Нет глобального горизонтального скролла: подтверждено (`documentElement.scrollWidth ≤ innerWidth + 1` на 6 ширинах × 2 темы).
- Сторонний виджет сохранён и не влияет на библиотеку: подтверждено (матрица #64: шрифт Georgia, пунктирная рамка, собственная кнопка с радиусом 0; при этом `.ui-button` сохраняет радиус 12px и токенную тему; статический тест 18).

Дополнительные автоматические прогоны (сверх обязательных):

- скрипт `scripts/matrix.mjs` — 64 сценария: **после 64 OK / 0 FAIL** (до: 31 OK / 33 FAIL);
- скрипт `scripts/contrast.mjs` — WCAG 2.1 контраст 18 пар текст/фон × 2 темы: **36 / 36 OK** (минимальный фактический 5.00 при норме 4.5; у disabled ≥ 5.85 при ослабленной норме 3.0);
- скрипт `scripts/probe.mjs` — каскадные трассировки «до/после» (см. AUDIT_TEMPLATE.md).

## Доказательства

| Проверка | Команда/сценарий | Результат | Скриншот/ссылка |
|---|---|---|---|
| Контраст уведомлений | `scripts/contrast.mjs`: info/success/warning/danger × light/dark | 8.56–14.33 (min 4.5) — OK | вывод скрипта в логе; `after_375_dark.png` |
| Фокус переключателя | Tab → `#switch-autosave` → Space (e2e + матрица #39–#41) | фокус принимается, outline 3px на треке, Space переключает | `after_switch_focus.png` |
| Действительно модальное окно | `#open-dialog` click → `:modal=true` → Escape → фокус на триггере (e2e + #45–#46) | OK | `after_dialog_light.png`, `after_dialog_dark.png` |
| Контракт кнопки | матрица #1–#13: варианты/размеры/hover/focus/disabled в 2 темах | OK (один базовый блок, все цвета — токены) | `after_1440_light.png` |
| Длинный текст карточки | 320/375 px, заголовок 70+ символов, вложенный alert (матрица #21–#24) | без overflow, текст alert не уменьшен | `after_320_light.png` |
| Токены тем | смена темы меняет accent/canvas (e2e + #61) | rgb(54,89,217)→rgb(143,165,254) — OK | `after_1440_dark.png` |
| Валидация после blur | e2e «field does not display error before user interacts» | untouched — без ошибки; после blur — aria-invalid=true | `after_field_error.png` |
| Vendor-изоляция | матрица #64 + тесты 02/03/18 | OK | `before_1440_light.png` vs `after_1440_light.png` |

Скриншоты: `reports/screenshots/before/` (6 файлов) и `reports/screenshots/after/` (16 файлов: 6 ширин × 2 темы + 2 диалога + фокус переключателя + поле).

## Известные ограничения

1. **Порог наведения hover не тестируется в 320px.** Псевдокласс `:hover` на touch-устройствах эмулируется браузером по-разному; матрица проверяет hover только на десктопных ширинах. План: прогон через `@media (hover: hover)`-обёртку для состояний, зависящих от указателя.
2. **Контраст disabled-элементов ниже 4.5.** После `opacity: .45` эффективный контраст текста disabled-кнопки падает ниже нормы для обычного текста (WCAG 1.4.3 исключает неактивные компоненты UI). Мы проверяем контраст до наложения opacity (≥ 5.85); альтернатива — отдельная пара токенов `--color-*-disabled`.
3. **Visual regression по пикселям не автоматизирован.** Скриншоты сравнивались человеком и через computed styles, а не попиксельным диффом (в задании это факультатив). План: `npx playwright test --update-snapshots` + `toHaveScreenshot()` с порогом 0.02.

## До/после

Наиболее показательный дефект — **A-01/A-16: primary-кнопка не управлялась темой и библиотекой** (vendor `#2939a8` перекрашивал все кнопки в `#workbench`).

Diff:

```css
/* app.css: было */
@import url('./legacy.css');                       /* unlayered — сильнее всех слоёв */
/* app.css: стало */
@import url('./legacy.css') layer(legacy);          /* ранний слой — слабее components */

/* legacy.css: было */
#workbench .ui-button { background: #2939a8; padding: .45rem .65rem; }
#workbench .ui-button.is-danger { background: #a41e24 !important; }
/* legacy.css: стало (область ограничена виджетом подрядчика) */
.legacy-widget button { border-radius: 0; }

/* components.css: было */
.ui-button--primary { background: #3659d9; color: #ffffff; }
/* components.css: стало */
.ui-button--primary { background: var(--color-accent); color: var(--color-accent-ink); }
```

Причина устранена архитектурно: unlayered-приоритет исчез (vendor в раннем слое, его глобальные селекторы ограничены `.legacy-widget`), вариант кнопки выражен семантическим токеном, поэтому тема, hover и disabled управляются заменой значений переменных без единого селектора в `components.css`. Каскад-трассировка до/после — в AUDIT_TEMPLATE.md, цепочка 1: unlayered 1-1-0 побеждал слой components 0-1-0; после — поздний слой components при normal сильнее раннего legacy, специфичность не участвует.
