```java

select jira_key,
       count(*) as relations,
       count(distinct agent_id) as agents,
       array_agg(distinct agent_id) as agent_ids
from jira_issue
where type = 'task'
  and lower(project) = 'crossgoal'
  and agent_id is not null
  and jira_key in (
      'CROSSGOAL-998910', 'CROSSGOAL-998911', 'CROSSGOAL-998912',
      'CROSSGOAL-998913', 'CROSSGOAL-998914', 'CROSSGOAL-998915',
      'CROSSGOAL-998916', 'CROSSGOAL-998917', 'CROSSGOAL-998918'
  )
group by jira_key
order by jira_key;

```
