```java

/**
 * GIVEN кэш TB_ALL пуст,
 * WHEN вызываем getAllBanks() дважды подряд,
 * THEN бэкенд (BaseMasterDataRequestService#requestData) дергается ровно один раз,
 *      результат попадает в кэш TerBankService2.TB_ALL под ключом "ALL",
 *      последующий вызов берёт данные из кэша.
 *
 * Проверяется:
 *  - имя кэша: {@code TerBankService2.TB_ALL};
 *  - ключ: {@code "ALL"};
 *  - отсутствие вызовов requestDataWithAttribute(...).
 */

 /**
 * GIVEN кэш TB_REQ_ALL пуст,
 * WHEN вызываем getAllBanksWithRequisite() дважды подряд,
 * THEN бэкенд (BaseMasterDataRequestService#requestDataWithAttribute) дергается ровно один раз,
 *      результат попадает в кэш TerBankService2.TB_REQ_ALL под ключом "ALL",
 *      последующий вызов берёт данные из кэша.
 *
 * Проверяется:
 *  - имя кэша: {@code TerBankService2.TB_REQ_ALL};
 *  - ключ: {@code "ALL"};
 *  - отсутствие вызовов requestData(...).
 */

 /**
 * Проверяет независимость двух кэшей.
 *
 * GIVEN по разу прогреты TB_ALL и TB_REQ_ALL,
 * WHEN очищаем только TB_ALL,
 * THEN содержимое TB_REQ_ALL остаётся нетронутым, а TB_ALL становится пустым.
 *
 * Дополнительно убеждаемся, что native-карты разных кэшей — разные инстансы.
 */

 /**
 * Конкурентные вызовы для "без атрибутов" сходятся в один backend-запрос.
 *
 * GIVEN несколько параллельных вызовов getAllBanks(),
 * WHEN включён sync=true и первый запрос к бэкенду выполняется с задержкой,
 * THEN BaseMasterDataRequestService#requestData(...) вызывается ровно один раз,
 *      остальные потоки получают тот же результат (после прогрева — из кэша TB_ALL/"ALL").
 */

 /**
 * Конкурентные вызовы для "с атрибутами" сходятся в один backend-запрос.
 *
 * GIVEN несколько параллельных вызовов getAllBanksWithRequisite(),
 * WHEN включён sync=true и первый запрос к бэкенду выполняется с задержкой,
 * THEN BaseMasterDataRequestService#requestDataWithAttribute(...) вызывается ровно один раз,
 *      остальные потоки получают тот же результат (после прогрева — из кэша TB_REQ_ALL/"ALL").
 */

 /**
 * Универсальный ассерт «fan-in в один backend-вызов» для конкурентных сценариев.
 *
 * <p>Последовательность:
 * <ol>
 *   <li>Выполняется заглушка бэкенда (обычно with Thread.sleep внутри thenAnswer).</li>
 *   <li>Параллельно запускаются N одинаковых вызовов «метода под тестом».</li>
 *   <li>Проверяем контракт: валидацию взаимодействий с бэкендом и наличие ключа в ожидаемом кэше.</li>
 * </ol>
 *
 * @param <T>                       тип возвращаемого значения callUnderTest (например, List<TerBankDto>)
 * @param stubBackend               раннер, который настраивает mock-ответ бэкенда (thenAnswer/thenReturn)
 * @param callUnderTest             Callable, выполняющий метод SUT (например, terBankCacheOps::getAllBanks)
 * @param verifyBackendInteractions проверка, что бэкенд вызван ровно один раз и альтернативные методы не дергались
 * @param assertCacheName           имя кэша, в котором должен появиться ключ "ALL"
 * @throws Exception пробрасывается, если один из потоков или проверок завершился с ошибкой
 * @see CacheIntrospection
 */

 /**
 * Подготавливает окружение теста:
 *  - мокает SearchRequestProperties и BaseMasterDataRequestService#getProperties();
 *  - задаёт успешные ответы (OK) для requestData(...) и requestDataWithAttribute(...);
 *  - настраивает мапперы terBankMapper и terBankWithRequisiteMapper.
 *
 * Использует фикстуры из MasterDataFixtures и вспомогательные стабы:
 * {@code stubRequestDataOk(...)} и {@code stubRequestDataWithAttrOk(...)}.
 */

 /**
 * Сброс тестового состояния между кейсами.
 * Очищает кэши TB_ALL и TB_REQ_ALL и обнуляет историю вызовов Mockito-моков.
 *
 * Гарантирует, что каждый тест стартует «с чистого листа».
 */

```
