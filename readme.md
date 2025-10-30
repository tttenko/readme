```java

/**
 * Возвращает список контрагентов по критериям (ИНН и/или КПП).
 * <p>
 * Кэширование:
 * <ul>
 *   <li><b>ИНН и КПП заданы</b> — используется регион {@code supplier_by_inn_kpp},
 *       ключ формата {@code inn:%s:kpp:%s}; данные загружаются через батч-механику
 *       {@link BatchCacheSupport#fetchBatch(String, java.util.List, java.util.function.Function, java.util.function.Function, Class)}.</li>
 *   <li><b>Задан только один из параметров</b> — выполняется прямая загрузка из сервиса
 *       (без батчей), после чего результат <b>вручную сохраняется в кэш</b> того же региона
 *       по ключу {@code inn:%s:kpp:%s} для последующих обращений.</li>
 * </ul>
 * </p>
 *
 * @param inn ИНН контрагента (опционально)
 * @param kpp КПП контрагента (опционально)
 * @return успешный результат со списком {@link CounterpartyDto}
 * @see #buildCriteriaMap(String, String)
 * @see #buildInnKppKey(String, String)
 * @since 1.0
 */
```
