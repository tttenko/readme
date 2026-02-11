```java

@JacksonXmlRootElement(localName = "PutEODPriceN")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNDto implements FxEnvelope {

  @JacksonXmlProperty(localName = "RQUID")
  @JsonDeserialize(converter = XmlConverters.ToUuidStrict.class)
  public UUID rquid;

  @JacksonXmlProperty(localName = "RqTm")
  @JsonDeserialize(converter = XmlConverters.ToLocalDateTimeLenientStrict.class)
  public LocalDateTime rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;

  @Override public UUID getRquid() { return rquid; }
  @Override public LocalDateTime getRqtm() { return rqtm; }
  @Override public List<FxRateXmlDto> getFxRates() { return fxRates; }
}

@JacksonXmlRootElement(localName = "FXRates")
@JsonIgnoreProperties(ignoreUnknown = true)
public class FxRateXmlDto {

  @JacksonXmlProperty(localName = "IsPublic")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isPublic;

  @JacksonXmlProperty(isAttribute = true, localName = "FXRateSubType")
  @JsonDeserialize(converter = XmlConverters.UpperTrimToNull.class)
  public String fxRateSubType;

  @JacksonXmlProperty(isAttribute = true, localName = "Code1")
  @JsonDeserialize(converter = XmlConverters.UpperTrimToNull.class)
  public String code1;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum1")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isoNum1;

  @JacksonXmlProperty(isAttribute = true, localName = "Code2")
  @JsonDeserialize(converter = XmlConverters.NormalizeRubUpper.class)
  public String code2;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum2")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isoNum2;

  // ВАЖНО: lenient -> null если не парсится
  @JacksonXmlProperty(isAttribute = true, localName = "UseDate")
  @JsonDeserialize(converter = XmlConverters.ToLocalDateLenientOrNull.class)
  public LocalDate useDate;

  @JacksonXmlProperty(isAttribute = true, localName = "LotSize")
  @JsonDeserialize(converter = XmlConverters.ToBigDecimalLenient.class)
  public BigDecimal lotSize;

  @JacksonXmlProperty(isAttribute = true, localName = "Value")
  @JsonDeserialize(converter = XmlConverters.ToBigDecimalLenient.class)
  public BigDecimal value;
}


```
