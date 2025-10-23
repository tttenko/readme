```java
// провайдер == Caffeine -> атомарная вычислялка без обёрток Spring Cache
    if (cache instanceof org.springframework.cache.caffeine.CaffeineCache caffeine) {
        @SuppressWarnings("unchecked")
        T one = (T) caffeine.getNativeCache().get(k, _unused ->
            safeFirst(loader.apply(List.of(k)))  // доменная ошибка полетит «как есть»
        );
        return (one == null) ? List.of() : List.of(one);
    }

```
