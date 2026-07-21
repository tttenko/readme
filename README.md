```java


PUT /api/v1/reference/metrics/{metricId}/pre-analytics

metricId:

10000000-0000-0000-0000-000000000001

Body:

{
  "code": "Точность",
  "isPreAnalytics": true
}

Ожидаемый статус:

200 OK
4. Заполни остальные метрики
CSI

metricId:

10000000-0000-0000-0000-000000000002
{
  "code": "Δ удовлетворённости",
  "isPreAnalytics": true
}
Охват

metricId:

10000000-0000-0000-0000-000000000003
{
  "code": "Охват пользователей",
  "isPreAnalytics": true
}
Скорость

metricId:

10000000-0000-0000-0000-000000000004
{
  "code": "Скорость",
  "isPreAnalytics": true
}

Каждый запрос должен вернуть 200 OK.

5. Проверь данные в БД
SELECT
    id,
    name,
    code,
    is_pre_analytics
FROM metrics_directory
WHERE id IN (
    '10000000-0000-0000-0000-000000000001',
    '10000000-0000-0000-0000-000000000002',
    '10000000-0000-0000-0000-000000000003',
    '10000000-0000-0000-0000-000000000004'
)
ORDER BY id;

Должны быть четыре строки с заполненным code и:

is_pre_analytics = true
6. Вызови pre-analytics
GET /api/v1/ai-agent/initiatives/320/pre-analytics

Ожидаемый результат:

errorCode = null;
в metricsAutonomous снова присутствуют четыре метрики;
если у инициативы есть copilot, они также находятся в metricsCopilot;
metricId, value, targetValue и deltaValue не изменились;
code теперь берётся из metrics_directory.

По твоим данным ожидаются:

metricId	code	value	targetValue	deltaValue
...0001	Точность	120	150	20
...0002	Δ удовлетворённости	80	150	-20
...0003	Охват пользователей	-2	150	33
...0004	Скорость	-2	150	-33
7. Докажи, что конфигурация больше не используется

На локальном стенде временно измени код четвёртой метрики:

{
  "code": "Скорость DB TEST",
  "isPreAnalytics": true
}

Повтори GET:

GET /api/v1/ai-agent/initiatives/320/pre-analytics

В ответе должно появиться:

"code": "Скорость DB TEST"

Затем отключи эту метрику:

{
  "code": "Скорость DB TEST",
  "isPreAnalytics": false
}

После повторного GET метрика «Скорость» должна исчезнуть из pre-analytics.

В конце обязательно восстанови:

{
  "code": "Скорость",
  "isPreAnalytics": true
}
```
