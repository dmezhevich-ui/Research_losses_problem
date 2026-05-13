Использовать как готовую целевую метрику
losses_sum_actual — основной факт потерь
refund_rate = total_refund_sum / compensation_sum — считать в сводной
compensation_sum / amount_buy — считать в сводной
losses_sum_actual / amount_buy — считать в сводной

**Короткий вывод**

Ниже — компактный список основных гипотез по процессу и таймингам.  
Для каждой гипотезы я укажу:

1. **смысл гипотезы**  
2. **что нужно для проверки**  
3. **на что смотреть в двух плоскостях**:
   - где **теряем больше денег** → `losses_sum_actual`
   - где **дороже решаем инцидент относительно стоимости брони** → `compensation_sum / amount_buy`

---

# Базовые метрики для всех гипотез

## Использовать в сводных
- `sum(losses_sum_actual)` — где реально больше потерь
- `sum(compensation_sum)` — где больше тратим на решение
- `sum(amount_buy)` — база для относительных коэффициентов
- `sum(total_refund_sum)` — для recovery

## Вычисляемые метрики в сводных
1. **Refund rate**  
   `sum(total_refund_sum) / sum(compensation_sum)`

2. **Compensation to booking ratio**  
   `sum(compensation_sum) / sum(amount_buy)`

3. **Loss to booking ratio**  
   `sum(losses_sum_actual) / sum(amount_buy)`

4. **FOC share**  
   доля `FOC` в `foc_status_actual`, `foc_status_1w`, `foc_status_2w`

---

# Основные гипотезы

---

## 1. Наличие причека снижает потери и удешевляет решение инцидента
### Гипотеза
Если по брони был `confirmation ticket`, то проблему либо предотвращают, либо выявляют раньше, поэтому:
- `losses_sum_actual` ниже
- `compensation_sum / amount_buy` ниже

### Поля
- `has_confirmation_ticket`

### Что нужно досчитать
Ничего обязательно не нужно.  
Только метрики в сводной:
- `refund_rate`
- `compensation_to_booking_ratio`
- `loss_to_booking_ratio`

### Что смотреть
#### Если хотим понять, где теряем больше денег
- `sum(losses_sum_actual)`
- `loss_to_booking_ratio`
- `refund_rate`
- `FOC share`

#### Если хотим понять, где дороже решаем инцидент
- `compensation_to_booking_ratio`

### Хорошо
- с причеком ниже `loss_to_booking_ratio`
- с причеком ниже `compensation_to_booking_ratio`
- выше `refund_rate`
- выше `FOC share`

### Плохо
- наличие причека не даёт улучшения
- или кейсы с причеком даже дороже  
Тогда либо причек делается поздно, либо он срабатывает уже на самых рискованных кейсах.

---

## 2. Причек до истечения free cancellation уменьшает потери
### Гипотеза
Если причек был сделан **до** дедлайна бесплатной отмены, компания успевает предотвратить дорогой сценарий.

### Поля
- `fok_to_confirmation_diff_days`

### Что нужно досчитать
Ничего, поле уже есть.  
Опционально сделать бакеты:
- `< 0`
- `0`
- `1–3`
- `4+`
- `null`

### Что смотреть
#### Потери
- `sum(losses_sum_actual)`
- `loss_to_booking_ratio`
- `refund_rate`
- `FOC share`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- при `< 0` ниже потери и ниже ratio
- выше `refund_rate`
- выше доля `FOC`

### Плохо
- после дедлайна free cancellation потери и расходы резко выше

---

## 3. Компенсация до истечения free cancellation уменьшает потери
### Гипотеза
Если первая компенсация была до окончания окна бесплатной отмены, кейс успели обработать в контролируемом сценарии.

### Поля
- `fok_to_compensation_diff_days`

### Что нужно досчитать
Ничего обязательно не нужно.  
Опционально — бакеты, как выше.

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`
- `FOC share`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- отрицательные значения дают более низкие потери и ниже ratio

### Плохо
- если компенсация выплачивается уже после FOC deadline и там заметно выше убыток

---

## 4. Чем быстрее назначается ответственный, тем меньше потери
### Гипотеза
Медленное назначение owner ухудшает outcome: меньше времени на эскалацию, договорённость, переигрывание кейса.

### Поля
- `ticket_to_owner_assigned_diff_days`

### Что нужно досчитать
Ничего обязательно.  
Опционально бакеты:
- `0`
- `1`
- `2–3`
- `4+`

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- чем меньше лаг, тем ниже потери и расходы относительно брони

### Плохо
- при длинных лагах растёт `loss_to_booking_ratio` и `compensation_to_booking_ratio`

---

## 5. Чем быстрее после назначения owner выплачивается первая компенсация, тем дешевле исход
### Гипотеза
Быстрое действие после назначения помогает не довести кейс до более дорогого решения.

### Поля
- `owner_assigned_to_compensation_diff_days`

### Что нужно досчитать
Ничего.  
Опционально бакеты:
- `0`
- `1`
- `2–3`
- `4+`

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- меньший лаг → меньшие потери и ниже ratio

### Плохо
- длинный лаг связан с более дорогими кейсами  
Нюанс: быстрые компенсации могут встречаться и в тяжёлых кейсах, это гипотеза с риском обратной причинности.

---

## 6. Длинный booking window снижает потери
### Гипотеза
Если между бронированием и заездом много времени, у команды больше шансов предотвратить дорогой сценарий.

### Поля
- `order_to_arrival_diff_days`

### Что нужно досчитать
Ничего.  
Опционально бакеты:
- `0–3`
- `4–7`
- `8–14`
- `15–30`
- `31+`

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `FOC share`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- длинный booking window → ниже потери и дешевле решение

### Плохо
- короткий booking window даёт самый высокий `loss_to_booking_ratio` и `compensation_to_booking_ratio`

---

## 7. Чем раньше после бронирования возникает тикет, тем дешевле можно решить проблему
### Гипотеза
Если проблема выявляется рано после создания заказа, есть больше пространства для недорогого решения.

### Поля
- `order_to_ticket_diff_days`

### Что нужно досчитать
Ничего.  
Опционально бакеты:
- `0`
- `1`
- `2–7`
- `8–30`
- `30+`

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- ранние тикеты дешевле

### Плохо
- поздние тикеты обходятся дороже и хуже отбиваются возвратами

---

## 8. Реально превентивный причек работает лучше, чем реактивный
### Гипотеза
Если `confirmation ticket` создан **до** incident ticket, то это настоящая профилактика. Если после — он уже реактивный.

### Поля
- `confirmation_to_ticket_diff_days`

### Что нужно досчитать
Ничего.  
Важно правильно интерпретировать знак:
- нужно использовать ту логику, что уже зафиксирована в поле:
  - **отрицательное значение** = тикет инцидента был раньше причека  
  - значит это плохо / реактивно

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- когда причек был раньше инцидента, потери ниже

### Плохо
- реактивный причек связан с высоким убытком и высокой стоимостью решения

---

## 9. Refundable брони дешевле в урегулировании, чем non-refundable
### Гипотеза
Если есть окно бесплатной отмены, у компании больше инструментов снизить ущерб.

### Поля
- `free_cancellation_before`

### Что нужно досчитать
**Да, тут полезно досчитать флаг:**
- `is_refundable`
  - `1` — `free_cancellation_before` не пустое
  - `0` — пустое

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`
- `FOC share`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- refundable кейсы дешевле и имеют лучший refund/FOC outcome

### Плохо
- non-refundable брони дают существенно более дорогой сценарий

---

## 10. Кейсы, обнаруженные ближе к arrival, обходятся дороже
### Гипотеза
Чем ближе инцидент к заезду, тем меньше времени на дешёвое решение.

### Поля
- `arrival_date`
- `ticket_created_date`

### Что нужно досчитать
**Да, здесь стоит досчитать:**
- `ticket_before_arrival_days = arrival_date - ticket_created_date`

Или сразу бакет:
- `>7 days before arrival`
- `1–7 days before arrival`
- `same day`
- `after arrival`

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- чем раньше до заезда создан тикет, тем дешевле outcome

### Плохо
- кейсы “в день заезда / после заезда” самые дорогие

---

## 11. FOC связан с меньшими потерями и меньшей дороговизной решения
### Гипотеза
Кейсы со статусом `FOC` должны быть финансово лучше, чем `Not FOC`.

### Поля
- `foc_status_actual`
- `foc_status_1w`
- `foc_status_2w`

### Что нужно досчитать
Ничего.

### Что смотреть
#### Потери
- `losses_sum_actual`
- `loss_to_booking_ratio`
- `refund_rate`

#### Дороговизна решения
- `compensation_to_booking_ratio`

### Хорошо
- `FOC` и особенно ранний `FOC` дают меньше потерь

### Плохо
- `Not FOC` системно дороже  
Это будет ожидаемо, но важно количественно измерить.

---

# Какие поля реально нужно досчитать в основном датасете

Если брать только то, что действительно нужно, а не “на всякий случай”, то список короткий:

## Обязательно
1. **`is_refundable`**
   - по `free_cancellation_before`

2. **`ticket_before_arrival_days`**
   - `arrival_date - ticket_created_date`

## Опционально, но удобно для анализа
3. Бакеты для:
- `order_to_arrival_diff_days`
- `order_to_ticket_diff_days`
- `ticket_to_owner_assigned_diff_days`
- `owner_assigned_to_compensation_diff_days`
- `fok_to_confirmation_diff_days`
- `fok_to_compensation_diff_days`
- `confirmation_to_ticket_diff_days`
- `ticket_before_arrival_days`

Но их можно делать уже в BI / SQL / pandas на этапе сводных.

---

# Какие гипотезы самые приоритетные

Если не хочется распыляться, я бы начал с 6:

1. `has_confirmation_ticket`
2. `fok_to_confirmation_diff_days`
3. `fok_to_compensation_diff_days`
4. `ticket_to_owner_assigned_diff_days`
5. `ticket_before_arrival_days`
6. `foc_status_actual / foc_status_1w / foc_status_2w`

Потому что именно они лучше всего отвечают на вопрос:
- **где мы теряем больше денег**
- и **где мы слишком дорого решаем кейс относительно стоимости брони**
