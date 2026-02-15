```java

@GetMapping(value = "/currencyRate", produces = APPLICATION_JSON_VALUE)
    @Operation(operationId = "getCurrencyRate", summary = "Получение курса по параметрам (code1, code2, date)")
    ResultObj<CurrencyRateDto> getCurrencyRate(
            @RequestParam("date")
            @NotNull
            @DateTimeFormat(pattern = "dd.MM.yyyy")
            LocalDate date,

            @RequestParam("code1")
            @NotBlank
            String code1,

            @RequestParam("code2")
            @NotBlank
            String code2
    );

@RestController
@RequiredArgsConstructor
public class CurrencyRateControllerImpl implements CurrencyRateController {

    private static final DateTimeFormatter OUT_DATE = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final CurrencyRateService currencyRateService;

    @Override
    public ResultObj<List<CurrencyRateDto>> getCurrencyRate(LocalDate date, String code1, String code2) {
        List<CurrencyRateDto> items = currencyRateService.getCurrencyRate(date, code1, code2);

        String description = "Найдено записей: %d на дату %s для пересчета из %s в %s"
                .formatted(items.size(), date.format(OUT_DATE), code1, code2);

        ResultObj<List<CurrencyRateDto>> result = new ResultObj<>();
        result.addMessages(ResultObjMessage.success("Поиск выполнен", null, description)); // <-- твой success(...)
        result.setData(items);
        result.setCount((long) items.size());
        return result;
    }
}

@@Service
@RequiredArgsConstructor
public class CurrencyRateService {

    private static final DateTimeFormatter OUT_DATE = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final FxRateRepository fxRateRepository;

    public List<CurrencyRateDto> getCurrencyRate(LocalDate date, String code1, String code2) {
        LocalDateTime useDateExclusive = date.plusDays(1).atStartOfDay();

        return fxRateRepository.findLatestRate(code1, code2, useDateExclusive)
                .map(e -> List.of(toDto(e)))
                .orElseGet(List::of);
    }

    private CurrencyRateDto toDto(FxRateEntity e) {
        BigDecimal value = e.getValue();
        BigDecimal lotSize = e.getLotSize();

        BigDecimal currencyRate = value.divide(lotSize, 4, RoundingMode.HALF_UP);

        return CurrencyRateDto.builder()
                .date(e.getUseDate().toLocalDate().format(OUT_DATE)) // дата найденного курса (может быть пятница)
                .value(value)
                .lotSize(lotSize)
                .currencyRate(currencyRate)
                .build();
    }
}

public interface FxRateRepository extends JpaRepository<FxRateEntity, Long> {

    @Query(value = """
        select *
        from fx_rate
        where code1 = :code1
          and code2 = :code2
          and use_date < :useDateExclusive
        order by use_date desc
        limit 1
        """, nativeQuery = true)
    Optional<FxRateEntity> findLatestRate(
            @Param("code1") String code1,
            @Param("code2") String code2,
            @Param("useDateExclusive") LocalDateTime useDateExclusive
    );
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Информация о курсе валюты/металла")
public class CurrencyRateDto implements Serializable {

    @Schema(description = "Дата в формате dd.MM.yyyy. Дата начиная с которой действует курс")
    private String date;

    @Schema(description = "Курс (Value) из источника")
    private BigDecimal value;

    @Schema(description = "Коэффициент (LotSize) из источника")
    private BigDecimal lotSize;

    @Schema(description = "Курс для 1 единицы: Value / LotSize (4 знака после запятой)")
    private BigDecimal currencyRate;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Информация о курсе валюты/металла")
public class CurrencyRateDto implements Serializable {

    @Schema(description = "Дата в формате dd.MM.yyyy. Дата начиная с которой действует курс")
    private String date;

    @Schema(description = "Курс (Value) из источника")
    private BigDecimal value;

    @Schema(description = "Коэффициент (LotSize) из источника")
    private BigDecimal lotSize;

    @Schema(description = "Курс для 1 единицы: Value / LotSize (4 знака после запятой)")
    private BigDecimal currencyRate;
}



```
