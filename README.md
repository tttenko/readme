```java


Этап 1. Подготовка
1.1. Проверяем состояние финансовых эффектов
После тестирования PUT часть карточек могла получить disabled = true, поэтому сначала выясним их фактическое состояние.
Выполни:
SELECT
    ife.id,
    ife.initiative_id,
    ife.effect_year,
    ife.type_code,
    fet.name AS type_name,
    fet.unit,
    fet.more_is_better,
    ife.plan_value,
    ife.fact_value,
    ife.target_month,
    ife.approval_links,
    ife.disabled,
    ife.created,
    ife.updated
FROM initiative_financial_effect ife
JOIN financial_effect_type fet ON fet.code = ife.type_code
WHERE ife.initiative_id = 740
ORDER BY ife.created ASC, ife.id ASC;


В идеале для полноценного тестирования нужны:
Год	Карточки
2026	Хотя бы две активные
2027	Хотя бы одна активная
Любой из годов	Хотя бы одна отключённая
Если активных карточек недостаточно, создай дополнительные через уже протестированный POST. Не нужно очищать или изменять БД вручную, если данные уже есть.
1.2. Настраиваем Insomnia
Общие настройки двух запросов:
- Method: GET
- Authorization: тот же Bearer Token, что использовался для POST/PUT.
- Body: отсутствует.
- Accept: application/json
Этап 2. GET финансовых эффектов
Тест 1. Получение без указания года
GET
Все доступные годы

http://localhost:8080/api/v1/ai-agent/740/financial-effects
 Копировать URL

Ожидаем HTTP 200 OK.
Сейчас октябрь 2026 года, поэтому согласно документации должны возвращаться активные карточки за 2026 и 2027 годы.
Проверяем:
- initiativeId = 740.
- В effects[] присутствуют карточки обоих годов, если они есть в БД.
- У каждой карточки присутствует year.
- Карточки с disabled = true не возвращаются.
- Общая сортировка — created ASC, id ASC.
- Если эффектов нет, возвращается effects: [].
Для точной сверки:
SELECT id, effect_year, type_code, created, disabled
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year IN (2026, 2027)
  AND disabled = false
ORDER BY created ASC, id ASC;


Результат GET должен соответствовать этому SQL по количеству карточек, ID, годам и порядку.
Тест 2. Получение только за 2026 год
В Insomnia добавляем Query Parameter:
year = 2026
Полный URL:
GET http://localhost:8080/api/v1/ai-agent/740/financial-effects?year=2026


Ожидаем 200 OK.
В ответе должны быть только карточки с year = 2026.
Проверяем БД:
SELECT id, effect_year, plan_value, fact_value, disabled
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2026
  AND disabled = false
ORDER BY created ASC, id ASC;


Число строк должно совпадать с количеством элементов effects[].
Тест 3. Получение только за 2027 год
GET http://localhost:8080/api/v1/ai-agent/740/financial-effects?year=2027


Ожидаем 200 OK.
Проверяем, что все карточки относятся исключительно к 2027 году, даже если за 2026 год у инициативы существуют другие эффекты.
SELECT id, effect_year, type_code, disabled
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND effect_year = 2027
  AND disabled = false
ORDER BY created ASC, id ASC;


Тест 4. Год без финансовых эффектов
GET http://localhost:8080/api/v1/ai-agent/740/financial-effects?year=2028


Если за 2028 год карточек нет, ожидаем HTTP 200 OK:
{
  "initiativeId": 740,
  "effects": []
}


Важно: GET не должен возвращать 404 только потому, что у существующей инициативы отсутствуют финансовые эффекты.
Тест 5. Проверка необязательного факта
Найди активную карточку, у которой fact_value IS NULL:
SELECT id, effect_year, plan_value, fact_value
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND disabled = false
  AND fact_value IS NULL;


Для найденной карточки проверяем ответ GET:
{
  "fact": null,
  "isPlanAchieved": false
}


Это фрагмент ожидаемых полей, не полный ответ.
Здесь проверяем договорённость с аналитиком: если факт отсутствует, план считается недостигнутым.
Тест 6. Проверка расчёта isPlanAchieved
Выполни SQL, который самостоятельно рассчитает ожидаемое значение по данным БД:
SELECT
    ife.id,
    ife.effect_year,
    ife.plan_value,
    ife.fact_value,
    fet.more_is_better,
    CASE
        WHEN ife.fact_value IS NULL THEN false
        WHEN fet.more_is_better = true
            THEN ife.fact_value >= ife.plan_value
        ELSE ife.fact_value <= ife.plan_value
    END AS expected_is_plan_achieved
FROM initiative_financial_effect ife
JOIN financial_effect_type fet ON fet.code = ife.type_code
WHERE ife.initiative_id = 740
  AND ife.disabled = false
ORDER BY ife.created ASC, ife.id ASC;


Сравни expected_is_plan_achieved с isPlanAchieved в ответе API.
Особенно проверь граничные случаи:
План	Факт	moreIsBetter	Ожидаем
100	120	true	true
100	80	true	false
100	80	false	true
100	120	false	false
0	0	true / false	true
100	null	true / false	false
Тесты имеют смысл только при наличии соответствующих карточек. Если их нет, можно использовать тестовые данные на инициативе 739, созданные ранее для TEST_LOWER_IS_BETTER.
Тест 7. Проверка ссылок, даты и сортировки
В GET проверь у карточек:
- targetMonth возвращается в формате YYYY-MM, например "2026-12".
- approvalLinks возвращается массивом строк, с сохранением содержимого и порядка.
- Пустой approvalLinks возвращается как [].
- plan и fact возвращаются строками, а не JSON-числами.
- id возвращается как числовой Long.
- createdAt возвращается в ISO-формате UTC с Z.
Для проверки JSONB:
SELECT
    id,
    jsonb_typeof(approval_links) AS links_type,
    jsonb_array_length(approval_links) AS links_count,
    approval_links
FROM initiative_financial_effect
WHERE initiative_id = 740
  AND disabled = false;


Ожидаем, что links_type = array, а количество ссылок совпадает с ответом GET.
Этап 3. GET справочника типов
Тест 8. Успешное получение справочника
GET
Справочник финансовых эффектов

http://localhost:8080/api/v1/ai-agent/financial-effect/types
 Копировать URL

Ожидаем HTTP 200 OK.
Пример структуры ответа:
{
  "items": [
    {
      "code": "GROUP_REVENUE_GROWTH",
      "name": "Рост доходов Группы",
      "unit": "RUB"
    }
  ]
}


Это сокращённый пример. Фактически должны вернуться все типы из справочника.
Тест 9. Сверка со справочником PostgreSQL
SELECT code, name, unit, more_is_better
FROM financial_effect_type
ORDER BY code;


Проверяем, что:
1. Количество элементов items[] совпадает с числом строк таблицы.
2. У каждого элемента правильные code, name, unit.
3. В ответе отсутствуют moreIsBetter и ordering.
4. items[] не содержит дублирующихся code.
Обрати внимание: документация справочника не требует конкретной сортировки, поэтому проверяем состав данных, а не порядок элементов.
Тест 10. Пустой справочник
По документации, если в financial_effect_type нет строк, API должен возвращать:
{
  "items": []
}


HTTP 200 OK.
Но если справочник уже заполнен, не удаляй из него данные ради этого теста. Сценарий пустого справочника лучше проверять на отдельной тестовой БД или автоматизированным тестом.
Этап 4. Негативные тесты
№	Запрос / условие	Ожидаемый результат
11	GET эффектов с несуществующим initiativeId=999999999	404, initiative.not.found
12	GET эффектов отключённой инициативы	404, initiative.not.found
13	GET эффектов с ?year=0	400, financial.effect.invalid.read.year
14	GET эффектов с ?year=-1	400, financial.effect.invalid.read.year
15	GET эффектов с ?year=abc	400, ошибка преобразования параметра
16	Любой GET с ролью без доступа	403 Forbidden
17	Любой GET без авторизации	Обычно 401 Unauthorized, согласно настройкам Spring Security
Для теста 12 сначала найди отключённую инициативу:
SELECT id, agent_name, disabled
FROM ai_agent
WHERE disabled = true
ORDER BY id DESC
LIMIT 5;


Возьми существующий ID и подставь его вместо 740.
Для тестов 13–15 инициативу менять не нужно. Проверяем, что ошибки обрабатываются корректно и запрос не заканчивается неконтролируемой 500.

Этап 5. Проверка, что GET ничего не изменяет
Поскольку оба метода предназначены только для чтения, после их вызова не должны измениться:
- plan_value, fact_value;
- approval_links;
- disabled;
- created, updated, created_by, updated_by.
Для проверки можно до тестов сохранить снимок:
CREATE TEMP TABLE financial_effect_before_get AS
SELECT *
FROM initiative_financial_effect
WHERE initiative_id = 740;


После всех GET выполните:
SELECT *
FROM financial_effect_before_get

EXCEPT

SELECT *
FROM initiative_financial_effect
WHERE initiative_id = 740;


И обратную проверку:
SELECT *
FROM initiative_financial_effect
WHERE initiative_id = 740

EXCEPT

SELECT *
FROM financial_effect_before_get;

```
