---
name: avito-kpi-report
description: Авито KPI отчёт в 14:00, 18:00, 22:00 Самара → Telegram @CleverlyUno + @IS_eg
---

You are an Avito analyst. Collect KPI data and send a report to TWO Telegram recipients: @CleverlyUno AND @IS_eg.

## Step 1 — Get date, time and determine report date

Run bash:
```
TZ="Europe/Samara" date +%Y-%m-%d
TZ="Europe/Samara" date +%H
TZ="Europe/Samara" date +%H:%M
TZ="Europe/Samara" date -d "yesterday" +%Y-%m-%d
```
Save as $TODAY, $HOUR (integer), $TIME (HH:MM), $YESTERDAY.

**SHIFT LOGIC — decide report date and hours into shift:**

- If $HOUR == 0 (midnight — end-of-shift snapshot):
  → Set $REPORT_DATE = $YESTERDAY
  → Set $REPORT_TIME = "23:59"
  → Set hours_into_shift = 13
  → Note: **🏁 Итог смены**

- If $HOUR >= 11:
  → Set $REPORT_DATE = $TODAY
  → Set $REPORT_TIME = $TIME
  → Set hours_into_shift = $HOUR - 11

- Otherwise (HOUR 1–10): this task normally only runs at 14/18/22 Samara, so this case shouldn't occur. If it does, output "Смена не началась — отчёт не нужен." and STOP.

## Step 2 — Get browser tab
Use tabs_context_mcp (createIfEmpty: true) → save tabId. Resize window to 1400x900.

## Step 3 — Collect aggregate stats
Navigate to: `https://www.avito.ru/profile/statistics?dateFrom=$REPORT_DATE&dateTo=$REPORT_DATE`
Wait 3 seconds. If redirected to login page — output "❌ Авито: требуется авторизация" and STOP.

Run JS `document.body.innerText` to read:
- Просмотры, CR% (the percentage between Просмотры and Контакты blocks)
- Контакты total, Написали в чат, Откликнулись на скидку
- Цена контакта (В среднем Х₽ за контакт)
- Избранное
- Активные объявления count
- Расходы total, На объявления, Другие

## Step 4 — Collect per-listing data
Navigate to: `https://www.avito.ru/profile/pro/items?filters=%7B%22statistics%22%3A%7B%22from%22%3A%22$REPORT_DATE%22%2C%22to%22%3A%22$REPORT_DATE%22%7D%7D`
Wait 3 seconds. Run JS:
```js
window.scrollTo(0, document.body.scrollHeight);
const t = document.body.innerText;
`hidden:${(t.match(/Скрыто/g)||[]).length} loaded:${t.includes('из 21')}`;
```
Then read full page text in chunks (lines 0-200, 200-400, 400-600) to extract per-listing stats:
- For each listing: Name, Views, Contacts, Spend (₽), CR%, Status
- Format: Views = first number, Contacts = second number (after +delta), Spend and CR follow
- Identify anomalies: high spend (>100₽) with 0 contacts

## Step 5 — Calculate KPI

- hours_into_shift determined in Step 1
- expected_contacts = 25 × (hours_into_shift / 4)  [norm: 25 contacts per 4h from 11:00]

KPI thresholds:
- Contacts: ✅ if actual ≥ expected, ⚠️ if actual ≥ 0.6×expected, ❌ if below
- CR: ✅ >15%, ⚠️ 10-15%, ❌ <10%
- Cost per contact: ✅ <150₽, ⚠️ 150-200₽, ❌ >200₽
- Spend: ✅ <12 000₽, ⚠️ 12 000-18 000₽, ❌ >18 000₽

## Step 6 — Compose report text

If end-of-shift (HOUR==0), add "🏁 Итог смены" in the header.

```
📊 Отчёт Авито: $REPORT_DATE ($REPORT_TIME Самара) [🏁 Итог смены — if end-of-shift]

🔹 Воронка продаж
Просмотры: [N] | CR: [%]%
Контакты: [N] ([X] в чат + [Y] скидка)
В избранном: [N] | Цена контакта: [₽]

🔹 Объявления с активностью
1. [Название] — [N] конт. | CR [%]% | Активно ✅
2. ...
Остальные [N] — 0 конт. | Активно ✅

🔹 Финансы
Расход: [₽] (на объявления: [₽] | инструменты: [₽])

🔹 KPI (норма: 25 конт./4ч от 11:00 | CR>15% | цена<150₽ | расход<12к)
[✅/⚠️/❌] Контакты: [N] (ожидается ≥[expected] за [hours_into_shift]ч смены)
[✅/⚠️/❌] CR: [%]%
[✅/⚠️/❌] Цена контакта: [₽]
[✅/⚠️/❌] Расход: [₽]
🔕 Скрытых: [N]

⚡ Вывод: 1-2 предложения — аномалии (высокий расход без контактов), лидеры по CR и контактам.
```

## Step 7 — Send to @CleverlyUno (ONE attempt only)
Navigate to: `https://web.telegram.org/k/#@CleverlyUno`
Wait 5 seconds. Check that a chat input is present:
```js
const el = document.querySelector('.input-message-input[contenteditable]') || document.querySelector('[contenteditable="true"]');
el ? 'ready' : 'no input';
```
If input not found — log "❌ @CleverlyUno: input not found" and skip. Do NOT retry.

If ready:
```js
const el = document.querySelector('.input-message-input[contenteditable]') || document.querySelector('[contenteditable="true"]');
el.focus(); document.execCommand('insertText', false, REPORT_TEXT);
```
Wait 1 sec. Get send button coordinates and click via mouse (NOT JS .click()):
```js
const btn = document.querySelector('button.btn-send');
const r = btn?.getBoundingClientRect();
r;
```
Use computer tool left_click at (r.x + r.width/2, r.y + r.height/2).
Wait 2 sec. Take screenshot to verify sent.

## Step 8 — Send to @IS_eg (ONE attempt only)
Navigate to: `https://web.telegram.org/k/#@IS_eg`
Wait 5 seconds. Check input (same JS check). If not found — log "❌ @IS_eg: input not found" and skip.

If ready: insert REPORT_TEXT via JS insertText, click send via mouse coordinates.
Wait 2 sec. Take final screenshot to confirm delivery.

## Notes
- **CRITICAL: Each recipient gets exactly ONE send attempt. Never retry sending.**
- If Telegram requires login — stop and log "❌ Telegram: требуется авторизация"
- Use JS insertText only — never type character by character
- Always send via mouse click using computed button coordinates, NOT JS .click()
- Max 3 screenshots total
