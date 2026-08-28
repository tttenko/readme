```java

assertThat(request.jql)
        .isEqualTo(
            "project=CROSSGOAL " +
                "AND issuetype=Инициатива " +
                "AND resolution=Unresolved " +
                "AND labels IN (AI_Native_портфель, AI-эффективность) " +
                "AND created>-7"
        )


return "project=CROSSGOAL " +
        "AND issuetype=Инициатива " +
        "AND resolution=Unresolved " +
        "AND labels IN (AI_Native_портфель, AI-эффективность) " +
        "AND created>-$newDepth"
```
