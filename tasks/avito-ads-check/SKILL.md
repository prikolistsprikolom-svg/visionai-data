---
name: avito-ads-check
description: Авито — проверка скрытых объявлений каждые 2 часа (11–23 Самара) → алерт @CleverlyUno + @IS_eg если есть скрытые
cron: "0 11,13,15,17,19,21,23 * * *"
timezone: Europe/Samara
---

Ты — автоматический агент мониторинга объявлений на Авито. Проверяешь скрытые объявления и при необходимости шлёшь алерт в Telegram.

## Шаг 1 — Получи текущее время

```bash
TZ=Europe/Samara date +"%H:%M"
```

Сохрани как $TIME.

## Шаг 2 — Открой Авито

Используй `tabs_context_mcp` (createIfEmpty: true) → сохрани tabId. Измени размер окна: 1400×900.

Перейди на:
```
https://www.avito.ru/profile/pro/items
```

Подожди 4 секунды. Если редирект на `/login` — вывести "❌ Авито: требуется авторизация" и СТОП.

## Шаг 3 — Найди количество скрытых объявлений

Прочитай текст страницы через get_page_text. Найди вкладку/таб "Скрытые" и число рядом с ней.

Дополнительно запусти JS:
```javascript
// Ищем все табы и их текст
Array.from(document.querySelectorAll('[class*="tab"], [role="tab"], button'))
  .map(el => el.textContent.trim())
  .filter(t => t.length > 0 && t.length < 60 && /[А-Яа-я]/.test(t))
```

Правило определения $HIDDEN_COUNT:
- Найден текст "Скрытые 0" или таба "Скрытые" нет → $HIDDEN_COUNT = 0
- Найден текст "Скрытые N" где N > 0 → $HIDDEN_COUNT = N

**Если $HIDDEN_COUNT = 0 → СТОП. Ничего не отправлять. Завершить молча.**

## Шаг 4 — Составь текст алерта

```
⚠️ Авито: есть скрытые объявления! ($TIME Самара)
Скрытых объявлений: $HIDDEN_COUNT
Проверь и восстанови:
https://www.avito.ru/profile/pro/items
```

## Шаг 5 — Отправь @CleverlyUno (ОДИН раз, без повторов)

Перейди на:
```
https://web.telegram.org/k/#@CleverlyUno
```

Подожди 5 секунд. Проверь наличие поля ввода:
```javascript
const el = document.querySelector('.input-message-input[contenteditable]')
  || document.querySelector('[contenteditable="true"]');
el ? 'ready' : 'no input';
```

Если 'no input' → вывести "❌ @CleverlyUno: input not found" и пропустить.

Если 'ready':
1. Вставить текст алерта:
```javascript
const el = document.querySelector('.input-message-input[contenteditable]')
  || document.querySelector('[contenteditable="true"]');
el.focus();
document.execCommand('insertText', false, ALERT_TEXT);
```
2. Получить координаты кнопки отправки:
```javascript
const btn = document.querySelector('button.btn-send');
const r = btn?.getBoundingClientRect();
r;
```
3. Кликнуть мышью (computer tool `left_click`) по центру кнопки: (r.x + r.width/2, r.y + r.height/2).
4. Подождать 2 сек. Сделать скриншот для подтверждения.

## Шаг 6 — Отправь @IS_eg (ОДИН раз, без повторов)

Перейди на:
```
https://web.telegram.org/k/#@IS_eg
```

Подожди 5 секунд. Повтори ту же процедуру: проверь input → вставь текст → кликни кнопку отправки → скриншот.

## Финал

После отправки обоим — задача выполнена.

**Правила:**
- Максимум 3 скриншота
- Каждому получателю — ровно 1 попытка отправки, без ретраев
- Если объявления не скрыты (0) — молча завершить, не отправлять ничего
- JS insertText вместо посимвольного ввода
- Клик по кнопке только через mouse coordinates (computer tool), не через JS .click()
