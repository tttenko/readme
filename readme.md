```java

// Стабируем fetchData так, чтобы он просто вызывал реальный лоадер
    Mockito.lenient().doAnswer(invocation -> {
        String cacheName = invocation.getArgument(0);
        List<String> keys = invocation.getArgument(1);

        if (SupplierService2.SUPPLIER_BY_INN_KPP.equals(cacheName)) {
            // отрабатывает путь inn+kpp: ключи -> лоадер -> мастер-данные
            return loaderSupplierByInnKpp.fetchByKeys(keys);
        }

        // на всякий случай для других кешей — пустой список
        return List.of();
    }).when(cacheGetOrLoadService)
      .fetchData(anyString(), anyList());

```
