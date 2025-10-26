```java

/**
 * <p><b>GIVEN</b> кэш <code>TB_ALL</code> пуст;<br>
 * <b>WHEN</b> вызываем <code>getAllBanks()</code> дважды;<br>
 * <b>THEN</b> бэкенд дергается ровно один раз, результат кэшируется
 * в <code>TerBankService2.TB_ALL</code> под ключом <code>"ALL"</code>, второй вызов читает из кэша.</p>
 *
 * <ul>
 *   <li>Проверяется имя кэша: <code>TerBankService2.TB_ALL</code>.</li>
 *   <li>Проверяется ключ кэша: <code>"ALL"</code>.</li>
 *   <li>Проверяется отсутствие вызовов <code>requestDataWithAttribute(...)</code>.</li>
 * </ul>
 */

/**
 * <p><b>GIVEN</b> кэш <code>TB_REQ_ALL</code> пуст;<br>
 * <b>WHEN</b> вызываем <code>getAllBanksWithRequisite()</code> дважды;<br>
 * <b>THEN</b> бэкенд дергается ровно один раз, результат кэшируется
 * в <code>TerBankService2.TB_REQ_ALL</code> под ключом <code>"ALL"</code>, второй вызов читает из кэша.</p>
 *
 * <ul>
 *   <li>Проверяется имя кэша: <code>TerBankService2.TB_REQ_ALL</code>.</li>
 *   <li>Проверяется ключ кэша: <code>"ALL"</code>.</li>
 *   <li>Проверяется отсутствие вызовов <code>requestData(...)</code>.</li>
 * </ul>
 */

/**
 * <p>Проверяет <b>независимость</b> кэшей <code>TB_ALL</code> и <code>TB_REQ_ALL</code>.</p>
 *
 * <p><b>GIVEN</b> оба кэша прогреты;<br>
 * <b>WHEN</b> очищаем только <code>TB_ALL</code>;<br>
 * <b>THEN</b> <code>TB_REQ_ALL</code> остаётся нетронутым, а <code>TB_ALL</code> пуст.</p>
 *
 * <p>Дополнительно убеждаемся, что underlying <code>ConcurrentMap</code> у кэшей — разные инстансы.</p>
 */

/**
 * <p>Конкурентные вызовы <code>getAllBanks()</code> при <code>sync=true</code>
 * сходятся в <b>один</b> вызов <code>BaseMasterDataRequestService#requestData(...)</code>.</p>
 *
 * <p><b>GIVEN</b> несколько параллельных вызовов и искусственная задержка первого бэкенд-ответа;<br>
 * <b>THEN</b> бэкенд вызывается один раз, дальнейшие чтения идут из кэша <code>TB_ALL/"ALL"</code>.</p>
 */

/**
 * <p>Конкурентные вызовы <code>getAllBanksWithRequisite()</code> при <code>sync=true</code>
 * сходятся в <b>один</b> вызов <code>BaseMasterDataRequestService#requestDataWithAttribute(...)</code>.</p>
 *
 * <p><b>GIVEN</b> несколько параллельных вызовов и искусственная задержка первого бэкенд-ответа;<br>
 * <b>THEN</b> бэкенд вызывается один раз, дальнейшие чтения идут из кэша <code>TB_REQ_ALL/"ALL"</code>.</p>
 */

/**
 * <p>Универсальная проверка «fan-in в один backend-вызов» для конкурентных сценариев.</p>
 *
 * <p><b>Алгоритм</b>:</p>
 * <ol>
 *   <li>Выполнить заглушку бэкенда (эмуляция задержки/ответа).</li>
 *   <li>Параллельно запустить N вызовов тестируемого метода.</li>
 *   <li>Проверить ожидаемые взаимодействия с бэкендом и наличие ключа <code>"ALL"</code> в нужном кэше.</li>
 * </ol>
 *
 * @param <T> тип результата <code>callUnderTest</code> (например, <code>List&lt;TerBankDto&gt;</code>)
 * @param stubBackend код, настраивающий мок бэкенда (обычно <em>thenAnswer</em> с задержкой)
 * @param callUnderTest вызов SUT (например, <code>terBankCacheOps::getAllBanks</code>)
 * @param verifyBackendInteractions валидация, что дернулся только ожидаемый метод бэкенда
 * @param assertCacheName имя кэша, где должен появиться ключ <code>"ALL"</code>
 * @throws Exception если любой из потоков или проверок завершился ошибкой
 * @see CacheIntrospection
 */

/**
 * <p>Подготавливает окружение теста.</p>
 * <ul>
 *   <li>Мокает <code>SearchRequestProperties</code> и <code>BaseMasterDataRequestService#getProperties()</code>.</li>
 *   <li>Задаёт успешные ответы (OK) для <code>requestData(...)</code> и <code>requestDataWithAttribute(...)</code>
 *       через фикстуры <code>MasterDataFixtures</code>.</li>
 *   <li>Настраивает мапперы <code>terBankMapper</code> и <code>terBankWithRequisiteMapper</code>.</li>
 * </ul>
 *
 * @implNote использует вспомогательные стабы:
 * <code>stubRequestDataOk(...)</code> и <code>stubRequestDataWithAttrOk(...)</code>.
 */

/**
 * <p>Сбрасывает состояние перед каждым тестом.</p>
 * <ul>
 *   <li>Очищает кэши <code>TB_ALL</code> и <code>TB_REQ_ALL</code>.</li>
 *   <li>Сбрасывает историю вызовов у Mockito-моков.</li>
 * </ul>
 *
 * <p>Гарантирует запуск каждого теста «с чистого листа».</p>
 */


```
