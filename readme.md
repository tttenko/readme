```java

/**
 * Сервис получения базовых ставок НДС с кешированием по дате и ставке.
 * <p>Работает поверх общего сервиса кеша {@link CacheGetOrLoadService}
 * и использует кеш с именем {@link #CACHE_NDS_BY_RATE}.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService2 {

    public static final String CACHE_NDS_BY_RATE = "nds_by_rate";

    private static final DateTimeFormatter DATE_TIME_FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

    private final CacheGetOrLoadService cacheGetOrLoadService;

    /**
     * Возвращает базовые ставки НДС с учётом фильтров по ставкам и кодам.
     * <p>Данные берутся из кеша {@link #CACHE_NDS_BY_RATE} либо подгружаются
     * через соответствующий {@code BatchLoader}, после чего фильтруются и
     * мапятся в {@link NdsDto}.
     *
     * @param date дата, на которую ищутся ставки; если {@code null}, используется текущая дата
     * @param code список кодов ставок НДС для фильтрации; {@code null} или пустой список означают,
     *             что фильтр по коду не применяется
     * @param rate список значений ставок НДС для фильтрации; {@code null} или пустой список означают,
     *             что фильтр по ставке не применяется
     * @return успешный результат с списком DTO ставок НДС
     */
    @NonNull
    public ResultObj<List<NdsDto>> getBasicVatRate(
            @Nullable final ZonedDateTime date,
            @Nullable final List<String> code,
            @Nullable final List<String> rate
    ) { ... }

    /**
     * Формирует список ключей кеша для заданной даты и набора ставок.
     * <p>Ключ имеет вид {@code <date>} либо {@code <date>|<rate>} для каждой ставки.
     * Пустые и {@code null} значения ставок отфильтровываются.
     *
     * @param date дата, на которую формируются ключи
     * @param rate список значений ставок; может быть {@code null}
     * @return список ключей кеша; как минимум один ключ с датой
     */
    @NonNull
    private List<String> buildKeys(@NonNull ZonedDateTime date,
                                   @Nullable List<String> rate) { ... }

    /**
     * Разворачивает бакеты ставок НДС в плоский список DTO и применяет фильтры.
     *
     * @param buckets список бакетов, полученных из кеша/лоадера
     * @param rate    список значений ставок для фильтрации; может быть {@code null}
     * @param code    список кодов ставок для фильтрации; может быть {@code null}
     * @return список DTO, удовлетворяющих заданным фильтрам
     */
    @NonNull
    private List<NdsDto> mapBucketsToDtoList(
            @NonNull List<RateBucket> buckets,
            @Nullable List<String> rate,
            @Nullable List<String> code
    ) { ... }

    /**
     * Строит предикат для фильтрации записей НДС по спискам ставок и кодов.
     * <p>Если соответствующий список {@code null} или пустой, фильтр по нему
     * не применяется.
     *
     * @param rate список значений ставок; может быть {@code null}
     * @param code список кодов ставок; может быть {@code null}
     * @return предикат для использования в стриме
     */
    @NonNull
    private Predicate<NdsFullDto> matches(
            @Nullable List<String> rate,
            @Nullable List<String> code
    ) { ... }

    /**
     * Маппит полную запись НДС в сокращённый DTO для ответа сервиса.
     *
     * @param e исходная запись из МД
     * @return DTO с основными полями ставки НДС
     */
    @NonNull
    private NdsDto toDto(@NonNull NdsFullDto e) { ... }

    /**
     * Полностью очищает кеш ставок НДС {@link #CACHE_NDS_BY_RATE}.
     * <p>Следующий вызов сервиса приведёт к повторной подгрузке данных через
     * {@code LoaderNdsByRate}.
     */
    @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
    public void cleanCache() {
        log.info("NDS cache cleared");
    }

    /**
     * Контейнер (бакет) для элементов НДС, сгруппированных по ключу кеша.
     *
     * @param key   ключ кеша, по которому хранятся данные
     * @param items набор записей НДС, соответствующих этому ключу
     */
    public record RateBucket(String key, Set<NdsFullDto> items) { }
}

```
