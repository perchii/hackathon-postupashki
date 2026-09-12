# MVP Measurement System — Поступашки

## Что это

Гибридный end-to-end prototype:

**placement → creative → tracking/deep link → touch → lead → REAL payment → attribution → ROMI_attr → forecast-ready mart**

MVP построен так, чтобы не генерировать downstream-продажи: оплаты, курсы, суммы и timestamps берутся из реального `base.xlsx`.

## Что REAL, а что DEMO

| Layer | Статус |
|---|---|
| Sales / payment | **REAL: `base.xlsx`** |
| Probable order count | **REAL**, группировка `student_id + timestamp` |
| Placement registry | **DEMO** |
| `creative_id` | **DEMO schema** |
| Tracking/deep link | **DEMO** |
| User touches | **SYNTHETIC** |
| Leads | **SYNTHETIC** |
| Historical source assignment | **SYNTHETIC / not claimed as fact** |
| Marketing cost | **SYNTHETIC** |
| Attributed revenue | Hybrid demo: real payment + demo source |
| ROMI_attr | Demo output of calculator |
| Forecast sales target | **REAL probable orders from `base.xlsx`** |

## Почему так

Исторический sales-layer наблюдаем, а user-level путь от рекламы до оплаты отсутствует.  
Поэтому MVP не пытается «восстановить» неизвестный source как факт.

Вместо этого он показывает, как система будет работать после внедрения tracking.

## Telegram stitching

Production flow:

```text
ad/post
  ↓
t.me/bot?start=<placement_id>_<creative_id>
  ↓
bot_start
  ↓
hashed Telegram user_id
  ↓
lead_id
  ↓
order/payment
```

Если переход идёт просто в Telegram-канал и user-level ID не передаётся, deterministic stitching невозможен. Такой трафик требует агрегатной оценки, promo/self-reported source или другого tracking-механизма.

## Data model в MVP

### Placement
- `campaign_id`
- `placement_id`
- `creative_id`
- `channel`
- `product_line`
- `published_at`
- `cost_rub`
- `tracking_payload`
- `tracking_link`

### Touch
- `touch_id`
- `user_id`
- `placement_id`
- `creative_id`
- `touch_time`

### Lead
- `lead_id`
- `user_id`
- `placement_id`
- `creative_id`
- `product_interest`
- `lead_created_at`
- `status`

### Payment
- `payment_id`
- `order_id`
- `lead_id`
- `user_id`
- `course`
- `amount_rub`
- `payment_time`

## Bundle ambiguity

`base.xlsx` содержит строки, которые могут относиться к bundle-заказам.

В MVP:
- probable order определяется как `student_id + exact timestamp`;
- для количества заказов используются все probable orders;
- для attributed revenue используются только single-row orders;
- bundle-candidates исключены из ROMI demo, чтобы не придумывать правило распределения выручки.

## Attribution

В версии v1 применяется simple single-source rule:

> payment получает source placement, сохранённый у соответствующего lead.

Production data model может хранить multiple touches и поддерживать first-touch / last-touch / linear / time-decay.

## Метрики

MVP считает:
- touches;
- leads;
- purchases;
- Touch → Lead conversion;
- Lead → Purchase conversion;
- CPL;
- CAC;
- attributed revenue;
- `ROMI_attr`.

Формула:

`ROMI_attr = (Attributed Revenue - Marketing Cost) / Marketing Cost`

Важно: в MVP source и cost synthetic, поэтому ROMI — демонстрация calculator, а не историческая оценка эффективности.

## Решение по бюджету

MVP не распределяет 300 000 ₽ пропорционально demo-ROMI.

Он показывает decision loop:

```text
measure
→ rank
→ reserve test budget
→ run holdout / A-B
→ update estimates
→ reallocate
```

## Forecast-ready mart

Forecast target строится из реального sales-layer:

- `orders`
- `buyers`
- `bundle_candidate_orders`

Marketing features пока demo:

- `demo_touches`
- `demo_leads`
- `demo_active_placements`
- `demo_unique_channels`
- `demo_planned_spend`

Также добавлены:
- `day_of_week`
- `orders_lag_1`
- `orders_roll_7`

После production tracking demo marketing features заменяются реальными.

## Что MVP доказывает

MVP доказывает, что proposed data model и pipeline технически связываются end-to-end:

**placement → user → lead → real payment → attribution → ROMI → forecast mart**

Он **не доказывает**, что конкретный исторический placement вызвал конкретную покупку.

## Основной notebook

- `notebooks/mvp_measurement_system_hybrid.ipynb`
