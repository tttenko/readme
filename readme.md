```java
/**
 * Преобразует набор значений и атрибутов мастер-данных в {@link CurrencyDto}.
 * <p>
 * Заполняет идентификатор и основные поля валюты (наименование, ISO-коды, код АСТ),
 * а также при возможности дополняет DTO символом валюты и его кодом на основании
 * {@link CurrencySymbolDictionary}.
 *
 * @param values базовые значения записи (например, идентификатор)
 * @param attributes карта атрибутов записи (значения по именам атрибутов)
 * @return заполненный {@link CurrencyDto}
 */

 /**
 * Маппер для преобразования DTO курса валют из XML в сущность {@link FxRateEntity}.
 * <p>
 * Используется для подготовки данных к сохранению, дополняя сущность служебными полями
 * (RQUID и RGTm) и перенося значения курса, кодов валют и параметров расчёта из {@link FxRateXmlDto}.
 */
@Component
public class FxRateEntityMapper {

    /**
     * Преобразует {@link FxRateXmlDto} в {@link FxRateEntity} и заполняет служебные поля записи.
     *
     * @param rquid уникальный идентификатор записи/сообщения
     * @param rgtm дата-время регистрации/получения записи
     * @param x DTO с данными курса валют, полученными из XML
     * @return заполненная сущность {@link FxRateEntity}
     */
    public FxRateEntity toEntity(String rquid, LocalDateTime rgtm, FxRateXmlDto x) { /* ... */ }

}

/**
 * DTO для входящего XML-сообщения {@code PutEODPriceNf}.
 * <p>
 * Содержит идентификатор запроса, время формирования и список курсов валют/металлов ({@code FXRates}).
 */
@JacksonXmlRootElement(localName = "PutEODPriceNf")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNfDto {

/**
 * JPA-сущность, представляющая запись курса валют в таблице {@code fx_rate}.
 * <p>
 * Хранит служебные поля записи (RQUID, RGTm), тип курса, коды и числовые ISO-коды валют,
 * дату применения курса, а также параметры курса (лот и значение).
 */
@Entity
@Setter
@Getter
@Table(name = "fx_rate")
public class FxRateEntity {
    // ...
}

/**
 * Репозиторий для работы с курсами валют ({@link FxRateEntity}).
 * <p>
 * Помимо стандартных CRUD-операций {@link JpaRepository} содержит:
 * <ul>
 *   <li>upsert (insert/update) записи курса по бизнес-ключу (subType, isoNum1, isoNum2, useDate)</li>
 *   <li>поиск последнего курса до указанной даты</li>
 * </ul>
 */
public interface FxRateRepository extends JpaRepository<FxRateEntity, Long> {

    /**
     * Выполняет upsert записи курса в таблицу {@code master_data_adapter.exchange_rate}.
     * <p>
     * Если запись с тем же бизнес-ключом {@code (fxratesubtype, isonum1, isonum2, use_date)} уже существует,
     * обновляет служебные поля и значения курса (коды валют, лот, значение).
     *
     * @param rquid уникальный идентификатор записи/сообщения
     * @param rgtm дата-время регистрации/получения записи
     * @param subType подтип курса
     * @param code1 буквенный код первой валюты
     * @param isoNum1 числовой ISO-код первой валюты
     * @param code2 буквенный код второй валюты
     * @param isoNum2 числовой ISO-код второй валюты
     * @param useDate дата применения курса
     * @param lotSize размер лота (если применимо)
     * @param value значение курса
     * @return количество затронутых строк (в зависимости от реализации БД/драйвера)
     */
    @Modifying
    @Query(/* ... */)
    int upsert(
        @Param("rquid") String rquid,
        @Param("rgtm") LocalDateTime rgtm,
        @Param("subType") String subType,
        @Param("code1") String code1,
        @Param("isoNum1") String isoNum1,
        @Param("code2") String code2,
        @Param("isoNum2") String isoNum2,
        @Param("useDate") LocalDateTime useDate,
        @Param("lotSize") BigDecimal lotSize,
        @Param("value") BigDecimal value
    );

    /**
     * Возвращает самый последний курс для пары валют до указанной даты (исключая её).
     * <p>
     * Ищет записи по {@code isoNum1/isoNum2} и выбирает запись с максимальной {@code use_date},
     * где {@code use_date < useDateExclusive}.
     *
     * @param isoNum1 числовой ISO-код первой валюты
     * @param isoNum2 числовой ISO-код второй валюты
     * @param useDateExclusive верхняя граница даты применения (исключающая)
     * @return найденный курс или {@link Optional#empty()}, если подходящих записей нет
     */
    @Query(/* ... */)
    Optional<FxRateEntity> findLatestRate(
        @Param("isoNum1") String isoNum1,
        @Param("isoNum2") String isoNum2,
        @Param("useDateExclusive") LocalDateTime useDateExclusive
    );
}


```
