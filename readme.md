```java

Mockito.lenient().doAnswer(invocation -> {
        String cacheName = invocation.getArgument(0);
        @SuppressWarnings("unchecked")
        List<String> keys = invocation.getArgument(1);

        if (AdapterService2.UOM_BY_CODE.equals(cacheName)) {
            return loaderUomByCode.fetchByKeys(keys);
        }
        if (AdapterService2.MATERIAL_TYPE_BY_ID.equals(cacheName)) {
            return loaderMaterialTypeById.fetchByKeys(keys);
        }
        if (AdapterService2.MATERIAL_BY_CODE.equals(cacheName)) {
            return loaderMaterialByCode.fetchByKeys(keys);
        }

        return List.of(); // на всякий случай
    }).when(cacheGetOrLoadService).fetchData(Mockito.anyString(), Mockito.anyList());
```
