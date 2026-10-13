```java


INSERT INTO financial_effect_type (
    code,
    name,
    unit,
    more_is_better
)
VALUES
    (
        'GROUP_REVENUE_GROWTH',
        'Рост доходов Группы',
        'RUB',
        true
    ),
    (
        'RELEASED_FTE_COUNT',
        'Количество высвобождаемых ПШЕ',
        'ПШЕ',
        true
    ),
    (
        'TEST_LOWER_IS_BETTER',
        'Тестовый эффект (меньше — лучше)',
        'RUB',
        false
    )
ON CONFLICT (code) DO NOTHING;
```
