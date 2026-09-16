```java

DO $$
DECLARE
    v_initiative_metric_type_id BIGINT := 37;
    v_metric_id UUID := '055b6470-af1e-41cd-a912-3440f3d7d3e5';
    v_user_id BIGINT := 123456;

    v_assignment_id BIGINT;
    v_first_request_id BIGINT;
    v_second_request_id BIGINT;
BEGIN

    INSERT INTO initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    VALUES (
        v_initiative_metric_type_id,
        v_metric_id,
        'NOT_APPLICABLE'
    )
    ON CONFLICT (initiative_agent_type_id, metric_id)
    DO UPDATE SET applicability_status = 'NOT_APPLICABLE'
    RETURNING id INTO v_assignment_id;


    /* Первый завершённый цикл.
       Специально is_visible_in_office=false:
       история всё равно обязана вернуть эту заявку. */
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
    VALUES (
        v_assignment_id,
        'APPROVED',
        'TEST_HISTORY_FIRST_REQUEST',
        DATE '2026-09-01',
        false,
        DATE '2026-08-01',
        DATE '2026-09-01',
        v_user_id,
        v_user_id,
        DATE '2026-08-15',
        TIMESTAMP '2026-09-01 01:00:00'
    )
    RETURNING id INTO v_first_request_id;


    INSERT INTO metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    VALUES (
        v_first_request_id,
        'REQUEST_CREATED',
        v_user_id::TEXT,
        'TEST_HISTORY_REQUEST_CREATED_1',
        DATE '2026-08-15'
    );


    INSERT INTO metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    VALUES (
        v_first_request_id,
        'APPROVED',
        v_user_id::TEXT,
        'TEST_HISTORY_APPROVED_1',
        DATE '2026-08-15'
    );


    INSERT INTO metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    VALUES (
        v_first_request_id,
        'RESUME_PERIOD',
        'SYSTEM',
        'TEST_HISTORY_RESUME_PERIOD',
        DATE '2026-09-01'
    );


    /* Второй, текущий цикл. */
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
    VALUES (
        v_assignment_id,
        'APPROVED',
        'TEST_HISTORY_SECOND_REQUEST',
        DATE '2026-12-01',
        true,
        DATE '2026-10-01',
        DATE '2026-12-01',
        v_user_id,
        v_user_id,
        DATE '2026-09-15',
        TIMESTAMP '2026-09-15 12:00:00'
    )
    RETURNING id INTO v_second_request_id;


    INSERT INTO metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    VALUES (
        v_second_request_id,
        'REQUEST_CREATED',
        v_user_id::TEXT,
        'TEST_HISTORY_REQUEST_CREATED_2',
        DATE '2026-09-15'
    );


    INSERT INTO metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    VALUES (
        v_second_request_id,
        'APPROVED',
        v_user_id::TEXT,
        'TEST_HISTORY_APPROVED_2',
        DATE '2026-09-15'
    );


    RAISE NOTICE 'assignmentId = %', v_assignment_id;
    RAISE NOTICE 'firstRequestId = %', v_first_request_id;
    RAISE NOTICE 'secondRequestId = %', v_second_request_id;

END $$;
```
