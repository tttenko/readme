```java


1. Создаём на DEV тестовую ситуацию как на PROD. Скрипт сам найдёт ACTIVE assignment типа copilot, у которого нет активной заявки и ещё нет значения за октябрь. Ничего подставлять не нужно.
WITH candidate AS (
    SELECT
        ima.id AS assignment_id,
        ima.initiative_agent_type_id,
        ima.metric_id,
        imt.ai_agent_id AS initiative_id,
        imt.agent_type
    FROM initiative_metric_assignment ima
    JOIN initiative_metric_type imt
        ON imt.id = ima.initiative_agent_type_id
    WHERE ima.applicability_status = 'ACTIVE'
      AND imt.agent_type = 'copilot'

      -- Не берём assignment с действующей заявкой
      AND NOT EXISTS (
          SELECT 1
          FROM metric_applicability_request mar
          WHERE mar.initiative_metric_assignment_id = ima.id
            AND mar.status IN ('PENDING', 'APPROVED')
            AND mar.is_visible_in_office = true
      )

      -- Не берём метрику, по которой уже есть значение за октябрь
      AND NOT EXISTS (
          SELECT 1
          FROM initiative_metric_value imv
          WHERE imv.initiative_metric_type_id = ima.initiative_agent_type_id
            AND imv.metric_directory_id = ima.metric_id
            AND imv.period_month = DATE '2026-10-01'
      )

    ORDER BY ima.id
    LIMIT 1
),
inserted_request AS (
    INSERT INTO metric_applicability_request (
        initiative_metric_assignment_id,
        status,
        comment,
        resume_period,
        is_visible_in_office,
        effective_from_period,
        effective_to_period,
        created_by,
        decision_by,
        created_at,
        updated_at
    )
    SELECT
        candidate.assignment_id,
        'APPROVED',
        'DEV_TEST_SUBMISSION_MONTH_20261119',
        DATE '2026-11-01',
        false,
        DATE '2026-10-01',
        DATE '2026-11-01',
        0,
        0,
        DATE '2026-10-19',
        TIMESTAMP '2026-11-01 01:00:00'
    FROM candidate
    RETURNING id, initiative_metric_assignment_id
)
SELECT
    c.initiative_id,
    c.agent_type,
    c.assignment_id,
    c.metric_id,
    ir.id AS request_id
FROM candidate c
JOIN inserted_request ir
    ON ir.initiative_metric_assignment_id = c.assignment_id;

В результате запиши себе:
initiative_id
agent_type
assignment_id
metric_id
request_id

Мы искусственно получили именно состояние:
effective_from = 01.10.2026
effective_to   = 01.11.2026
resume_period  = 01.11.2026

assignment = ACTIVE
request = APPROVED
visible = false

То есть: в октябре метрику не сдавали, с ноября снова сдаём. Это соответствует состоянию после отработки scheduler: он закрывает request по resumePeriod и возвращает assignment в ACTIVE.    Вставленный текст(20261119-0821…

Сразу можно проверить старую и новую логику одним SELECT:
SELECT
    mar.id AS request_id,
    imt.ai_agent_id AS initiative_id,
    imt.agent_type,
    ima.id AS assignment_id,
    ima.metric_id,
    ima.applicability_status,
    mar.status,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.resume_period,

    (
        mar.status IN ('PENDING', 'APPROVED')
        AND mar.effective_from_period <= DATE '2026-10-01'
        AND (
            mar.effective_to_period IS NULL
            OR DATE '2026-10-01' < mar.effective_to_period
        )
    ) AS blocks_old_logic,

    (
        mar.status IN ('PENDING', 'APPROVED')
        AND mar.effective_from_period <= DATE '2026-11-01'
        AND (
            mar.effective_to_period IS NULL
            OR DATE '2026-11-01' < mar.effective_to_period
        )
    ) AS blocks_new_logic

FROM metric_applicability_request mar
JOIN initiative_metric_assignment ima
    ON ima.id = mar.initiative_metric_assignment_id
JOIN initiative_metric_type imt
    ON imt.id = ima.initiative_agent_type_id

WHERE mar.comment = 'DEV_TEST_SUBMISSION_MONTH_20261119';

Ожидаем:
blocks_old_logic = true
blocks_new_logic = false

Это прямо демонстрирует исправляемый баг:
старая логика:
01.10 <= 01.10 < 01.11
→ блокируем

новая логика:
01.10 <= 01.11 < 01.11
→ false
→ не блокируем

Теперь берёшь initiative_id, metric_id, agent_type из первого SELECT и вызываешь ваш POST /metrics/value только с этой одной метрикой.

Условно:
{
  "metricsValues": [
    {
      "metricId": "<metric_id из SELECT>",
      "agentType": "copilot",
      "metricValue": 10,
      "targetValue": 20
    }
  ]
}

После исправления ожидаем успешное сохранение, потому что в ноябре уже разрешено сдавать данные за октябрь.

Дальше обязательно делаем негативный контроль. Превращаем ту же заявку в ситуацию, где метрику разрешено сдавать только с декабря:
UPDATE metric_applicability_request
SET resume_period = DATE '2026-12-01',
    effective_to_period = DATE '2026-12-01',
    is_visible_in_office = true,
    updated_at = now()
WHERE comment = 'DEV_TEST_SUBMISSION_MONTH_20261119';

UPDATE initiative_metric_assignment
SET applicability_status = 'NOT_APPLICABLE'
WHERE id = (
    SELECT initiative_metric_assignment_id
    FROM metric_applicability_request
    WHERE comment = 'DEV_TEST_SUBMISSION_MONTH_20261119'
);

Теперь смысл:
effective_from = 01.10
effective_to   = 01.12

сейчас ноябрь
→ метрика всё ещё неприменима
→ октябрьские данные в ноябре сдавать нельзя

Повторяем тот же POST.

Теперь обязательно должен прийти Metric unlink:
01.10 <= 01.11 < 01.12
→ true

Если получаем:
Сценарий 1: resume = ноябрь  → POST SUCCESS
Сценарий 2: resume = декабрь → Metric unlink

то изменение работает именно так, как требуется по бизнес-логике.

После тестирования DEV можно очистить:
DELETE FROM initiative_metric_value imv
USING initiative_metric_assignment ima,
      metric_applicability_request mar
WHERE mar.comment = 'DEV_TEST_SUBMISSION_MONTH_20261119'
  AND mar.initiative_metric_assignment_id = ima.id
  AND imv.initiative_metric_type_id = ima.initiative_agent_type_id
  AND imv.metric_directory_id = ima.metric_id
  AND imv.period_month = DATE '2026-10-01';

DELETE FROM metric_applicability_history
WHERE metric_applicability_request_id IN (
    SELECT id
    FROM metric_applicability_request
    WHERE comment = 'DEV_TEST_SUBMISSION_MONTH_20261119'
);

DELETE FROM metric_applicability_request
WHERE comment = 'DEV_TEST_SUBMISSION_MONTH_20261119';

UPDATE initiative_metric_assignment
SET applicability_status = 'ACTIVE'
WHERE id NOT IN (
    SELECT initiative_metric_assignment_id
    FROM metric_applicability_request
    WHERE status IN ('PENDING', 'APPROVED')
      AND is_visible_in_office = true
)
AND applicability_status = 'NOT_APPLICABLE';

Последний UPDATE я бы не запускал в таком широком виде. Лучше после первого SELECT сохранить конкретный assignment_id и восстановить только его:
UPDATE initiative_metric_assignment
SET applicability_status = 'ACTIVE'
WHERE id = <тестовый assignment_id>;

Так безопаснее и вообще не затрагивает остальные DEV-данные.

Если хочешь, следующим сообщением могу собрать тебе один готовый SQL-файл: SETUP → CHECK → переключение на негативный кейс → CLEANUP, где останется только последовательно запускать блоки.
```
