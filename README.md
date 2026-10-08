```java

jql = "project=CROSSGOAL AND issuetype=Task " +
    "AND updated>-${updateDepth}d " +
    "AND \"Epic Link\" IN (\"Мониторинг портфеля AI-Native\")"

jql = "project=CROSSGOAL AND issuetype=Инициатива " +
    "AND updated>-${updateDepth}d " +
    "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,\"AI-эффективность\")) " +
    "OR labels=ClassicML OR status=\"Отменено\")",
```
