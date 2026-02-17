```java
@JacksonXmlRootElement(localName = "FXRates")
@JsonIgnoreProperties(ignoreUnknown = true)
public class FxRateXmlDto {

    /** Тип/подтип курса из атрибута {@code FXRates@FXRateSubType} (например, {@code FXCB}). */
    @JacksonXmlProperty(isAttribute = true, localName = "FXRateSubType")
    public String fxRateSubType;

    /** Буквенный код исходной валюты из {@code FXRates@Code1} (например, {@code AUD}). */
    @JacksonXmlProperty(isAttribute = true, localName = "Code1")
    public String code1;

    /** ISO numeric код исходной валюты из {@code FXRates@ISOnum1} (например, {@code 036}). */
    @JacksonXmlProperty(isAttribute = true, localName = "ISONum1")
    public String isoNum1;

    /** Буквенный код целевой валюты из {@code FXRates@Code2} (например, {@code RUB}). */
    @JacksonXmlProperty(isAttribute = true, localName = "Code2")
    public String code2;

    /** ISO numeric код целевой валюты из {@code FXRates@ISOnum2} (например, {@code 643}). */
    @JacksonXmlProperty(isAttribute = true, localName = "ISONum2")
    public String isoNum2;

    /** Дата применения курса из {@code FXRates@UseDate}. */
    @JacksonXmlProperty(isAttribute = true, localName = "UseDate")
    public ZonedDateTime useDate;

    /** Размер лота (кол-во единиц исходной валюты) из {@code FXRates@LotSize}. */
    @JacksonXmlProperty(isAttribute = true, localName = "LotSize")
    public BigDecimal lotSize;

    /** Значение курса из {@code FXRates@Value} (стоимость лота {@code Code1} в {@code Code2}). */
    @JacksonXmlProperty(isAttribute = true, localName = "Value")
    public BigDecimal value;
}
```
