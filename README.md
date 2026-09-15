```java
-- ============================================================
-- ТЕСТОВЫЕ ДАННЫЕ ДЛЯ GET /metric-applicability-requests
-- ============================================================

-- Удали данные предыдущего тестового запуска
DELETE FROM metric_applicability_request
WHERE comment LIKE 'TEST_QUEUE_%';


-- ============================================================
-- 1. PENDING / PENDING
-- availableActions = [APPROVE, REJECT]
-- ============================================================

WITH test_data AS (
    SELECT
        initiative_metric_type.id AS initiative_agent_type_id,
        metrics_directory.id AS metric_id
    FROM initiative_metric_type
    CROSS JOIN metrics_directory
    WHERE NOT EXISTS (
        SELECT 1
        FROM initiative_metric_assignment
        WHERE initiative_metric_assignment.initiative_agent_type_id = initiative_metric_type.id
          AND initiative_metric_assignment.metric_id = metrics_directory.id
    )
    ORDER BY initiative_metric_type.id, metrics_directory.id
    LIMIT 1
),
created_assignment AS (
    INSERT INTO initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    SELECT
        initiative_agent_type_id,
        metric_id,
        'PENDING'
    FROM test_data
    RETURNING id
)
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
    id,
    'PENDING',
    'TEST_QUEUE_PENDING',
    DATE '2026-12-01',
    TRUE,
    DATE '2026-10-01',
    NULL,
    123456,
    NULL,
    CURRENT_DATE,
    CURRENT_TIMESTAMP
FROM created_assignment;


-- ============================================================
-- 2. APPROVED / NOT_APPLICABLE
-- availableActions = [CANCEL_DECISION]
-- ============================================================

WITH test_data AS (
    SELECT
        initiative_metric_type.id AS initiative_agent_type_id,
        metrics_directory.id AS metric_id
    FROM initiative_metric_type
    CROSS JOIN metrics_directory
    WHERE NOT EXISTS (
        SELECT 1
        FROM initiative_metric_assignment
        WHERE initiative_metric_assignment.initiative_agent_type_id = initiative_metric_type.id
          AND initiative_metric_assignment.metric_id = metrics_directory.id
    )
    ORDER BY initiative_metric_type.id, metrics_directory.id
    LIMIT 1
),
created_assignment AS (
    INSERT INTO initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    SELECT
        initiative_agent_type_id,
        metric_id,
        'NOT_APPLICABLE'
    FROM test_data
    RETURNING id
)
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
    id,
    'APPROVED',
    'TEST_QUEUE_APPROVED',
    DATE '2027-01-01',
    TRUE,
    DATE '2026-10-01',
    DATE '2027-01-01',
    123456,
    654321,
    CURRENT_DATE - 1,
    CURRENT_TIMESTAMP
FROM created_assignment;


-- ============================================================
-- 3. REJECTED / ACTIVE
-- availableActions = [CANCEL_DECISION]
-- ============================================================

WITH test_data AS (
    SELECT
        initiative_metric_type.id AS initiative_agent_type_id,
        metrics_directory.id AS metric_id
    FROM initiative_metric_type
    CROSS JOIN metrics_directory
    WHERE NOT EXISTS (
        SELECT 1
        FROM initiative_metric_assignment
        WHERE initiative_metric_assignment.initiative_agent_type_id = initiative_metric_type.id
          AND initiative_metric_assignment.metric_id = metrics_directory.id
    )
    ORDER BY initiative_metric_type.id, metrics_directory.id
    LIMIT 1
),
created_assignment AS (
    INSERT INTO initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    SELECT
        initiative_agent_type_id,
        metric_id,
        'ACTIVE'
    FROM test_data
    RETURNING id
)
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
    id,
    'REJECTED',
    'TEST_QUEUE_REJECTED',
    NULL,
    TRUE,
    DATE '2026-10-01',
    NULL,
    123456,
    654321,
    CURRENT_DATE - 2,
    CURRENT_TIMESTAMP
FROM created_assignment;


-- ============================================================
-- 4. APPROVED / ACTIVE, НО СКРЫТА ИЗ ОФИСА
--
-- В GET вообще не должна попасть,
-- потому что is_visible_in_office = false
-- ============================================================

WITH test_data AS (
    SELECT
        initiative_metric_type.id AS initiative_agent_type_id,
        metrics_directory.id AS metric_id
    FROM initiative_metric_type
    CROSS JOIN metrics_directory
    WHERE NOT EXISTS (
        SELECT 1
        FROM initiative_metric_assignment
        WHERE initiative_metric_assignment.initiative_agent_type_id = initiative_metric_type.id
          AND initiative_metric_assignment.metric_id = metrics_directory.id
    )
    ORDER BY initiative_metric_type.id, metrics_directory.id
    LIMIT 1
),
created_assignment AS (
    INSERT INTO initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    SELECT
        initiative_agent_type_id,
        metric_id,
        'ACTIVE'
    FROM test_data
    RETURNING id
)
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
    id,
    'APPROVED',
    'TEST_QUEUE_HIDDEN',
    NULL,
    FALSE,
    DATE '2026-08-01',
    DATE '2026-09-01',
    123456,
    654321,
    CURRENT_DATE - 3,
    CURRENT_TIMESTAMP
FROM created_assignment;
```
