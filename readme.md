```java

 Mockito.lenient().doAnswer(invocation -> {
        // 0-й аргумент – имя кэша
        String cacheName = invocation.getArgument(0, String.class);
        // 1-й аргумент – список ключей
        @SuppressWarnings("unchecked")
        List<String> keys = invocation.getArgument(1, List.class);

        // небольшие проверки на всякий случай
        assertNotNull(cacheName, "cacheName must not be null");
        assertNotNull(keys, "keys must not be null");

        if (CurrencyService2.CURRENCY_BY_CODE.equals(cacheName)) {
            // ведём себя так, как в реальном CacheGetOrLoadService:
            // делегируем в лоадер
            return loaderCurrencyByCode.fetchByKeys(keys);
        }

        // если вдруг кто-то в тесте позвал кэш с другим именем — сразу падать
        fail("Unexpected cacheName in stub: " + cacheName);
        return List.of(); // для компилятора, реально не дойдём
    }).when(cacheGetOrLoadService).fetchData(Mockito.anyString(), Mockito.anyList());
```
