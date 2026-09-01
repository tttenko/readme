```java
select *
from agent_strategy
where ai_agent_id = 739;

select *
from contact
where lower(email) = lower('NIBubentsova@sberbank.ru');

select *
from agent_contact
where agent_id = 739;

select *
from jira_issue
where agent_id = 739;
```
