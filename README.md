```java


Этап 1. Подготовка
1.1. Проверяем состояние БД после POST
Выполни:
SELECT
    id,
    initiative_id,
    effect_year,
    type_code,
    plan_value,
    fact_value,
    target_month,
    approval_links,
    disabled,
    created,
    updated,
    updated_by
FROM initiative_financial_effect
WHERE initiative_id = 740
ORDER BY effect_year, created, id;


Если все тесты POST успешно выполнены, у инициативы должны быть три активные карточки за 2026 год и одна за 2027 год.
Нам нужны реальные ID трёх карточек за 2026 год.
Обозначение	Тип	План
ID_A	GROUP_REVENUE_GROWTH	10 000 000
ID_B	GROUP_REVENUE_GROWTH	5 000 000
ID_C	RELEASED_FTE_COUNT	0
Во всех запросах ниже я буду использовать примерные ID 101, 102, 103. Замени их на реальные значения из своей БД.
1.2. Настраиваем Insomnia
PUT
Сохранение финансовых эффектов

URL
http://localhost:8080/api/v1/ai-agent/740/financial-effects/2026

 Копировать
Headers: Content-Type: application/json и тот же Authorization, который использовался для POST.

Главное правило PUT: передаём полный актуальный список карточек. Если не передать ID существующей карточки, Backend воспримет это как команду удалить её через disabled = true.

Этап 2. Негативные тесты
Все негативные тесты лучше выполнить до успешного PUT, чтобы первоначальные три карточки оставались неизменными.
Тест 1. Год ещё не инициализирован
Если за 2025 год у инициативы 740 нет вообще никаких карточек, включая отключённые, отправь:
PUT /api/v1/ai-agent/740/financial-effects/2025
{
  "effects": []
}


Ожидаем:
400 Bad Request
financial.effect.not.initialized


Обоснование: PUT разрешён только для года, за который ранее уже сохраняли финансовые эффекты.
Тест 2. Несуществующий ID карточки
Верни URL на 2026 год.
{
  "effects": [
    {
      "id": 999999999,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "10000000",
      "fact": "12000000",
      "targetMonth": "2026-12",
      "approvalLinks": []
    }
  ]
}


Ожидаем 400:
financial.effect.not.found


Карточки 101, 102, 103 должны остаться активными. Несмотря на то что они отсутствуют в запросе, валидация должна остановить операцию до Soft Delete.
Тест 3. Попытка изменить тип сохранённой карточки
Для этого используем ID_A, у которого исходный тип GROUP_REVENUE_GROWTH.
{
  "effects": [
    {
      "id": 101,
      "typeCode": "RELEASED_FTE_COUNT",
      "plan": "10000000",
      "fact": "12000000",
      "targetMonth": "2026-12",
      "approvalLinks": []
    }
  ]
}


Ожидаем 400:
financial.effect.immutable.type


Проверяем, что в БД у карточки 101 остался прежний type_code.
Тест 4. Повторяющийся ID в запросе
{
  "effects": [
    {
      "id": 101,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "10000000",
      "targetMonth": "2026-12"
    },
    {
      "id": 101,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "15000000",
      "targetMonth": "2026-11"
    }
  ]
}


Ожидаем 400:
financial.effect.duplicate.id


Одна карточка не должна обновляться дважды в рамках одного PUT.
Тест 5. Невалидная карточка среди валидных
{
  "effects": [
    {
      "id": 101,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "15000000",
      "fact": "14000000",
      "targetMonth": "2026-12"
    },
    {
      "id": 102,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "-5000000",
      "targetMonth": "2026-11"
    }
  ]
}


Ожидаем 400:
financial.effect.invalid.amount


Самая важная проверка: план у карточки 101 должен остаться 10000000, а карточка 103 не должна отключиться.
Выполни:
SELECT id, plan_value, disabled, updated
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
ORDER BY id;


Все три карточки должны остаться в первоначальном состоянии.
Тест 6. ID принадлежит другому году
Из предыдущего тестирования POST у нас есть карточка за 2027 год.
Возьми её реальный ID из БД и попробуй передать его в PUT за 2026 год.
Ожидаем:
400 Bad Request — financial.effect.not.found.
Так мы убедимся, что нельзя обновить карточку другого года через неправильный URL.

Этап 3. Happy Path — создание, обновление и удаление одновременно
Это главный тест PUT.
Мы должны одним запросом выполнить сразу четыре действия.
Карточка	Действие	Ожидаемый результат
ID_A (101)	Изменить план, факт и ссылки	UPDATE
ID_B (102)	Оставить без изменений	No-op
ID_C (103)	Не передать в запросе	Soft Delete
Без ID	Добавить новую карточку	INSERT
Тест 7. Полное сохранение финансовых эффектов
PUT /api/v1/ai-agent/740/financial-effects/2026
Body → JSON:

{
  "effects": [
    {
      "id": 101,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "15000000",
      "fact": "13000000",
      "targetMonth": "2026-12",
      "approvalLinks": [
        "https://example.com/approval/1",
        "https://example.com/approval/updated"
      ]
    },
    {
      "id": 102,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "5000000",
      "targetMonth": "2026-11",
      "approvalLinks": []
    },
    {
      "typeCode": "RELEASED_FTE_COUNT",
      "plan": "10.5",
      "fact": "8.25",
      "targetMonth": "2026-12",
      "approvalLinks": [
        "https://example.com/approval/new"
      ]
    }
  ]
}



200 OK
Ожидаемый результат

После сохранения Backend должен вернуть полный актуальный список из трёх карточек: 101, 102 и новой с автоматически созданным ID.
Карточки 103 в ответе быть не должно.

Проверяем PostgreSQL
SELECT
    id,
    effect_year,
    type_code,
    plan_value,
    fact_value,
    target_month,
    approval_links,
    disabled,
    created,
    created_by,
    updated,
    updated_by
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
ORDER BY created ASC, id ASC;


Ожидаем:
Карточка	disabled	updated	updated_by
101	false	Заполнено	Текущий пользователь
102	false	NULL	NULL
103	true	Заполнено	Текущий пользователь
Новая	false	NULL	NULL
Предполагается, что до PUT карточки ещё не редактировались.
Также важно проверить, что у 101 и 102 не изменилось поле created.
Проверка количества:
SELECT
    COUNT(*) AS total_count,
    COUNT(*) FILTER (WHERE disabled = false) AS active_count,
    COUNT(*) FILTER (WHERE disabled = true) AS disabled_count
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026;


4
Всего строк

3
Активные

1
Отключённая




Этап 4. Проверяем отсутствие лишних обновлений
Тест 8. Повторный PUT с теми же значениями
Сначала сохрани текущие значения updated и updated_by для всех трёх активных карточек.
Затем отправь тот же набор из теста 7, но новой карточке обязательно поставь её реальный ID, полученный из ответа теста 7.
После повторного PUT ожидаем:
- HTTP 200 OK.
- Количество карточек не изменилось.
- Не появилось новых строк.
- updated и updated_by остались прежними.
Для сравнения:
SELECT id, plan_value, fact_value, updated, updated_by
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
  AND disabled = false
ORDER BY id;


Почему это важно: в новом FinancialEffectUpdateService есть проверка изменения значений. Если пользователь ничего не поменял, Hibernate не должен отправлять лишний UPDATE.
Тест 9. Изменение только ссылок
Теперь у одной существующей карточки измени только approvalLinks, оставив plan, fact и targetMonth прежними.
В PUT обязательно передай остальные активные карточки без изменений.
Ожидаем:
- Только изменённая карточка получает новое updated.
- У неё меняется updatedBy.
- Остальные карточки не обновляются.
- В PostgreSQL сохраняется новый JSON-массив ссылок.
Это дополнительно подготовит проверку историчности, когда подключим Hibernate Envers.

Этап 5. Удаление всех карточек
Тест 10. Пустой массив
Этот тест выполняй после остальных, поскольку он отключит все активные карточки за 2026 год.
PUT /api/v1/ai-agent/740/financial-effects/2026
{
  "effects": []
}


Ожидаем:
{
  "initiativeId": 740,
  "effects": []
}


HTTP 200 OK.
Проверяем БД:
SELECT id, disabled, updated, updated_by
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
ORDER BY id;


Все строки за 2026 год должны иметь disabled = true.
Физически записи не удаляются.
Тест 11. Повторный PUT с пустым массивом
Отправь тот же запрос ещё раз:
{
  "effects": []
}


Ожидаем 200 OK и effects: [].
updated у ранее отключённых карточек не должен измениться.
Это проверяет важный сценарий: после Soft Delete всех карточек PUT должен продолжать работать, поскольку первоначальное сохранение этого года уже происходило.

Этап 6. Проверка сохранения после удаления всех карточек
Тест 12. Добавление новой карточки через PUT
Когда все карточки отключены, отправь:
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "25000000",
      "fact": "20000000",
      "targetMonth": "2026-12",
      "approvalLinks": []
    }
  ]
}


Ожидаем 200 OK.
Backend должен создать новую карточку, не восстанавливая старые автоматически.
Проверь:
SELECT id, type_code, plan_value, disabled, created
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
ORDER BY id;


Ожидаем, что старые карточки остались disabled = true, а новая — disabled = false.

```
