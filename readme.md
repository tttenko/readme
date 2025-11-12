```java

/** Кладёт в hitsOut все найденные в кэше элементы. Некорректные ключи пропускает. */
    private <T> void collectHits(@Nullable Cache cache,
                                 @NonNull List<String> keys,
                                 @NonNull Class<T> type,
                                 @NonNull Map<String, T> hitsOut) {
        if (cache == null) return;
        for (String key : keys) {
            if (key == null || key.isBlank()) {
                log.debug("Skip invalid cache key: '{}'", key);
                continue;
            }
            T cached = cache.get(key, type);
            if (cached != null) {
                hitsOut.put(key, cached);
            }
        }
    }

    /**
     * Возвращает промахи: валидные ключи, которых нет среди hits.
     * (Поведение такое же, как раньше: невалидные ключи НЕ считаем миссами.)
     */
    @NonNull
    private List<String> collectMisses(@NonNull List<String> keys,
                                       @NonNull Map<String, ?> hits) {
        List<String> miss = new ArrayList<>();
        for (String key : keys) {
            if (key == null || key.isBlank()) continue; // как и раньше — пропускаем
            if (!hits.containsKey(key)) miss.add(key);
        }
        return miss;
    }

```
