```java


Коллеги, нужно проверить состояние данных по применимости метрик на PROD для инициативы, на которой при сохранении значений возвращается ошибка `400 / Metric unlink`.

Что означает `Metric unlink` в текущей реализации:

при вызове `POST /metrics/value` backend сохраняет значения за предыдущий месяц. Перед сохранением он проверяет, не была ли хотя бы одна из переданных метрик неприменима в этом периоде.

Для этого backend ищет по `initiative_metric_assignment` связанные записи `metric_applicability_request` со статусом `PENDING` или `APPROVED`, у которых сохраняемый месяц входит в интервал:

`effective_from_period <= saving_period`

и одновременно:

`effective_to_period IS NULL`
или
`saving_period < effective_to_period`.

Если хотя бы одна такая запись найдена, весь POST отклоняется с `Metric unlink`.

То есть `Metric unlink` фактически означает: backend считает, что среди переданных метрик есть метрика, которую в сохраняемом месяце нельзя было сдавать из-за действующей неприменимости.

Сейчас подозрение именно на состояние данных в БД.

Нормальная логика состояний следующая:

— при создании заявки создаётся `PENDING` request, а assignment переводится в `PENDING`;
— при согласовании request становится `APPROVED`, assignment — `NOT_APPLICABLE`;
— если задан `resume_period`, при APPROVE `effective_to_period` должен быть заполнен этим же значением;
— после RESTORE или окончания `resume_period` assignment возвращается в `ACTIVE`, а интервал неприменимости должен быть закрыт через `effective_to_period`.

Поэтому подозрительными являются, например, такие комбинации:

`assignment = ACTIVE`
+
`request = APPROVED`
+
`effective_to_period = NULL`

В этом случае текущий статус assignment уже говорит, что метрика активна, но историческая заявка выглядит для backend как незакрытый период неприменимости «с effective_from_period и дальше без конца».

Из-за этого при сохранении значений за предыдущий месяц backend находит такую заявку и возвращает `Metric unlink`, даже если на UI сама метрика уже выглядит активной.

Аналогично подозрительно состояние:

`assignment = ACTIVE`
+
`request = PENDING`
+
`is_visible_in_office = true`

Потому что активная PENDING-заявка должна соответствовать `assignment = PENDING`.

На DEV такие неконсистентные состояния уже находили, но там часть данных могла ранее создаваться/изменяться вручную SQL-скриптами, поэтому DEV не является чистым доказательством дефекта кода.

На ИФТ сценарий повторно проверили на чистых данных, созданных только через текущий UI/API — дефект не воспроизвёлся.

Поэтому сейчас нужно проверить PROD по конкретной проблемной инициативе и понять, нет ли там старых/неконсистентных записей в связке:

`initiative_metric_type`
→ `initiative_metric_assignment`
→ `metric_applicability_request`.

Если на PROD найдём, что assignment уже `ACTIVE`, но при этом остаётся `PENDING`/`APPROVED` request с интервалом, который перекрывает сохраняемый месяц, это объяснит `Metric unlink` именно состоянием данных.

После этого отдельно нужно будет определить, каким штатным действием эти данные были сформированы: старой версией логики, scheduler, RESTORE/CANCEL_DECISION либо другим переходом состояния.


WITH params AS (
    SELECT (date_trunc('month', current_date) - interval '1 month')::date AS saving_period
)
SELECT
    p.saving_period,

    imt.ai_agent_id AS initiative_id,
    imt.agent_type,

    ima.id AS assignment_id,
    ima.metric_id,
    md.name AS metric_name,
    ima.applicability_status,

    mar.id AS request_id,
    mar.status AS request_status,
    mar.resume_period,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.is_visible_in_office,
    mar.created_at,
    mar.updated_at,

    CASE
        WHEN mar.status IN ('PENDING', 'APPROVED')
             AND mar.effective_from_period <= p.saving_period
             AND (
                 mar.effective_to_period IS NULL
                 OR p.saving_period < mar.effective_to_period
             )
        THEN true
        ELSE false
    END AS blocks_metric_value

FROM params p
CROSS JOIN initiative_metric_type imt
JOIN initiative_metric_assignment ima
    ON ima.initiative_agent_type_id = imt.id
JOIN metrics_directory md
    ON md.id = ima.metric_id
LEFT JOIN metric_applicability_request mar
    ON mar.initiative_metric_assignment_id = ima.id

WHERE imt.ai_agent_id = :initiativeId

ORDER BY
    ima.id,
    mar.created_at DESC,
    mar.id DESC;
```
