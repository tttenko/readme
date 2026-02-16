```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Информация о курсе валюты/металла")
public class CurrencyRateDto {

    @Schema(description = "Исходная валюта/драгоценный металл")
    @JsonProperty("code1")
    private String code1;

    @Schema(description = "ISO код исходной валюты")
    @JsonProperty("ISONum1")
    private String isoNum1;

    @Schema(description = "Целевая валюта")
    @JsonProperty("code2")
    private String code2;

    @Schema(description = "ISO код целевой валюты")
    @JsonProperty("ISONum2")
    private String isoNum2;

    @Schema(description = "Дата в формате dd.MM.yyyy. Дата начиная с которой действует курс")
    @JsonProperty("date")
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "dd.MM.yyyy")
    private LocalDate date;

    @Schema(description = "Курс (Value) из источника")
    @JsonProperty("Value")
    private BigDecimal value;

    @Schema(description = "Коэффициент (LotSize) из источника")
    @JsonProperty("LotSize")
    private BigDecimal lotSize;

    @Schema(description = "Курс для 1 единицы: Value / LotSize (4 знака после запятой)")
    @JsonProperty("currencyRate")
    private BigDecimal currencyRate;
}

return CurrencyRateDto.builder()
            .code1(e.getCode1())
            .isoNum1(e.getIsoNum1())
            .code2(e.getCode2())
            .isoNum2(e.getIsoNum2())
            .date(e.getUseDate().toLocalDate())
            .value(value)
            .lotSize(lotSize)
            .currencyRate(currencyRate)
            .build();



```
