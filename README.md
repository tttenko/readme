```java
1. PATCH APPROVE

Метод:

PATCH /api/v1/ai-agent/initiatives/1/metrics/10000000-0000-0000-0000-000000000002/copilot/applicability-requests/15

То есть к своему host добавь:

/api/v1/ai-agent/initiatives/1/metrics/10000000-0000-0000-0000-000000000002/copilot/applicability-requests/15

Body → JSON:

{
  "action": "APPROVE",
  "comment": "Тестовое согласование заявки"
}

Токен должен быть пользователя с ролью:

TRANSFORMATION_OFFICE
2. Что ожидаем в ответе

По бизнес-логике:

{
  "requestId": 15,
  "requestStatus": "APPROVED",
  "applicabilityStatus": "NOT_APPLICABLE",
  "availableActions": [
    "CANCEL_DECISION"
  ]
}

Если в твоём текущем DTO ещё присутствует pendingCount, он просто будет дополнительным полем ответа — это сейчас не мешает тестированию state machine.

3. Что должно поменяться в БД

После успешного 200 выполни:

SELECT
    r.id                              AS request_id,
    r.status                          AS request_status,
    a.applicability_status            AS applicability_status,
    r.decision_by,
    r.resume_period,
    r.effective_from_period,
    r.effective_to_period,
    r.is_visible_in_office,
    r.updated_at
FROM prm_ai.metric_applicability_request r
JOIN prm_ai.initiative_metric_assignment a
    ON a.id = r.initiative_metric_assignment_id
WHERE r.id = 15;

Ожидаем:

request_id             = 15
request_status         = APPROVED
applicability_status   = NOT_APPLICABLE
decision_by            = ID пользователя, которым ты вызвал PATCH
resume_period          = 2026-12-01
effective_from_period  = 2026-09-01
effective_to_period    = 2026-12-01
is_visible_in_office   = true

effective_to_period должен скопироваться из resume_period при APPROVE.

4. Проверяем history
SELECT
    h.id,
    h.metric_applicability_request_id AS request_id,
    h.action,
    h.created_by,
    h.comment,
    h.created_at
FROM prm_ai.metric_applicability_history h
WHERE h.metric_applicability_request_id = 15
ORDER BY h.created_at, h.id;

Должно быть две записи:

REQUEST_CREATED
APPROVED

У APPROVED:

created_by = ID текущего пользователя
comment    = Тестовое согласование заявки
5. Один нюанс с email

В тестовых данных мы указали:

created_by = 0

Поэтому после commit код попробует уведомить пользователя 0. Скорее всего в логах увидишь сообщение, что пользователь не найден. Для этого теста это нормально: состояние в БД уже должно сохраниться и откатываться из-за уведомления не должно.

Сделай PATCH APPROVE и пришли мне response из Insomnia. После этого проверим БД и сразу на этой же заявке протестируем CANCEL_DECISION.

```
