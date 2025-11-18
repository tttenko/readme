```java

 // первый вызов — все ставки по указанной дате
    checkResult(performGetOk(mockMvc,
            "/api/v1/main-nds-code?date=" + date), 6);

    // фильтр по коду
    checkResult(performGetOk(mockMvc,
            "/api/v1/main-nds-code?code=NOVAT&date=" + date), 1);

    // фильтр по коду и ставке
    checkResult(performGetOk(mockMvc,
            "/api/v1/main-nds-code?code=NOVAT&rate=0&date=" + date), 1);

    // проверка работы кеша
    service.cleanCache();
    checkResult(performGetOk(mockMvc,
            "/api/v1/main-nds-code?code=NOVAT&rate=0&date=" + date), 1);

```
