```java


ест 2. Не передан обязательный план
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.required.field.
Тест 3. Не передан тип эффекта
{
  "effects": [
    {
      "plan": "1000000",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.required.field.
Тест 4. Отрицательный план
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "-1000000",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.invalid.amount.
Тест 5. Отрицательный факт
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "1000000",
      "fact": "-500000",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.invalid.amount.
Тест 6. Некорректные числовые значения
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "abc",
      "fact": "100",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.invalid.amount.
Также проверь fact = "abc" с корректным планом.
Тест 7. Несуществующий тип эффекта
{
  "effects": [
    {
      "typeCode": "UNKNOWN_TYPE",
      "plan": "1000000",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 400, код financial.effect.type.not.found.
Тест 8. Некорректный месяц
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "1000000",
      "targetMonth": "2026-13"
    }
  ]
}


Ожидаем 400, код financial.effect.invalid.target.month.
Дополнительно проверь:
targetMonth	Ожидаемый результат
"2026-00"	400
"2026-1"	400
"2026-12-15"	400
null	400
"2026-12"	Валидно
Тест 9. Запрещённый год создания
Измени URL:
POST /api/v1/ai-agent/740/financial-effects/2028


Body:
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "1000000",
      "targetMonth": "2028-12"
    }
  ]
}


Ожидаем 400, код financial.effect.invalid.year.
Сейчас октябрь 2026 года, поэтому согласно требованиям можно создавать эффекты за 2026 и 2027 годы.
Тест 10. Несуществующая инициатива
Используй заведомо несуществующий ID:
POST /api/v1/ai-agent/999999999/financial-effects/2026


Body:
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "1000000",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем 404 Not Found, код initiative.not.found.
Тест 11. Одна валидная карточка, вторая невалидная
Это очень важный тест для проверки транзакционности.
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "1000000",
      "targetMonth": "2026-12"
    },
    {
      "typeCode": "RELEASED_FTE_COUNT",
      "plan": "-100",
      "targetMonth": "2026-11"
    }
  ]
}


Ожидаем:
- HTTP 400.
- Ни одна карточка не сохраняется.
Проверь БД:
SELECT COUNT(*)
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026;


Если изначально записей не было, результат должен быть 0.

Этап 3. Успешное создание
Теперь проверяем основной Happy Path.
Тест 12. Создание трёх карточек за 2026 год
URL:
POST /api/v1/ai-agent/740/financial-effects/2026


Body:
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "10000000",
      "fact": "12000000",
      "targetMonth": "2026-12",
      "approvalLinks": [
        "https://example.com/approval/1",
        "https://example.com/approval/2"
      ]
    },
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "5000000",
      "targetMonth": "2026-11"
    },
    {
      "typeCode": "RELEASED_FTE_COUNT",
      "plan": "0",
      "fact": "0",
      "targetMonth": "2026-10",
      "approvalLinks": []
    }
  ]
}


Ожидаемый HTTP-статус
201 Created

Ожидаемое поведение:
- Создаются три самостоятельные карточки.
- Две карточки имеют одинаковый typeCode — это разрешено.
- id у всех трёх карточек разные и имеют тип Long.
- У второй карточки fact = null, isPlanAchieved = false.
- Для первой карточки isPlanAchieved = true, если more_is_better = true.
- Для третьей карточки isPlanAchieved = true при plan = 0, fact = 0 и реализованном сравнении с равенством.
- Отсутствующий approvalLinks заменяется на [].

Пример структуры ответа (ID и время будут другими):
{
  "initiativeId": 740,
  "effects": [
    {
      "id": 1,
      "typeCode": "GROUP_REVENUE_GROWTH",
      "typeName": "Рост доходов Группы",
      "unit": "RUB",
      "plan": "10000000",
      "fact": "12000000",
      "targetMonth": "2026-12",
      "approvalLinks": [
        "https://example.com/approval/1",
        "https://example.com/approval/2"
      ],
      "isPlanAchieved": true,
      "createdAt": "2026-10-13T12:00:00Z"
    }
  ]
}


Это сокращённый пример первой карточки. Реальный ответ должен содержать все три.
Проверка в PostgreSQL
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
    created_by,
    updated,
    updated_by
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
ORDER BY created ASC, id ASC;


Ожидаем:
Поле	Результат
Количество строк	3
effect_year	2026
disabled	false
created	Заполнено
created_by	ID текущего пользователя
updated	NULL
updated_by	NULL
approval_links	Корректный JSON-массив
fact_value второй карточки	NULL
Проверка количества:
SELECT COUNT(*)
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
  AND disabled = false;


Результат должен быть 3.

Этап 4. Повторное создание
Тест 13. Повторный POST за 2026 год
Теперь отправь ещё раз запрос из теста 12.
Ожидаем:
400 Bad Request


Код ошибки:
financial.effect.already.initialized


Почему? Потому что у инициативы 740 уже есть активные финансовые эффекты за 2026 год.
После ошибки выполни:
SELECT COUNT(*)
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
  AND disabled = false;


Должно остаться 3 карточки, а не 6.

Этап 5. Создание следующего года
Тест 14. Создание карточки за 2027 год
URL:
POST /api/v1/ai-agent/740/financial-effects/2027


Body:
{
  "effects": [
    {
      "typeCode": "GROUP_REVENUE_GROWTH",
      "plan": "15000000",
      "fact": "5000000",
      "targetMonth": "2027-12",
      "approvalLinks": []
    }
  ]
}


Ожидаем HTTP 201 Created.
Проверяем:
SELECT
    effect_year,
    COUNT(*) AS effects_count
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND disabled = false
GROUP BY effect_year
ORDER BY effect_year;


Ожидаемый результат:
effect_year	effects_count
2026	3
2027	1
Таким образом подтверждаем, что эффекты за разные годы хранятся отдельно.

Этап 6. Проверка moreIsBetter = false
Для этого теста возьмём другую инициативу из твоей БД — 739, если у неё нет активных карточек за 2026 год.
Тест 15. Когда меньше — лучше
URL:
POST /api/v1/ai-agent/739/financial-effects/2026


Body:
{
  "effects": [
    {
      "typeCode": "TEST_LOWER_IS_BETTER",
      "plan": "100",
      "fact": "80",
      "targetMonth": "2026-12"
    },
    {
      "typeCode": "TEST_LOWER_IS_BETTER",
      "plan": "100",
      "fact": "120",
      "targetMonth": "2026-12"
    },
    {
      "typeCode": "RELEASED_FTE_COUNT",
      "plan": "10.5",
      "fact": "8.25",
      "targetMonth": "2026-12"
    }
  ]
}


Ожидаем HTTP 201.
Карточка	План	Факт	isPlanAchieved
TEST_LOWER_IS_BETTER	100	80	true
TEST_LOWER_IS_BETTER	100	120	false
RELEASED_FTE_COUNT	10.5	8.25	false
Третья карточка также проверяет сохранение дробных значений через BigDecimal.

Этап 7. Проверка JSONB, дат и сортировки
После успешных тестов выполни:
SELECT
    id,
    effect_year,
    type_code,
    plan_value,
    fact_value,
    jsonb_typeof(approval_links) AS links_type,
    jsonb_array_length(approval_links) AS links_count,
    created,
    disabled
FROM initiative_financial_effect
WHERE initiative_id IN (740, 739)
ORDER BY initiative_id, effect_year, created, id;


Проверяем, что:
- links_type = array.
- Количество ссылок совпадает с запросом.
- created заполнено.
- disabled = false.
- Порядок карточек соответствует created ASC, id ASC.
Для дополнительной проверки времени:
SELECT
    CURRENT_TIMESTAMP AT TIME ZONE 'UTC' AS utc_now,
    created
FROM initiative_financial_effect
WHERE initiative_id = 740
ORDER BY id DESC
LIMIT 1;


Значение created должно соответствовать UTC, если JVM приложения настроена на UTC. Само по себе преобразование в Instant при формировании ответа не исправляет время, записанное в другой зоне.
```
