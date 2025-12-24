```java
@Slf4j
@RestController
@RequiredArgsConstructor
public class NdsUiControllerImpl implements UiNdsController {

    private final NdsService ndsService;
    private final ResponseHandler responseHandler;

    @Override
    public ResultObj<List<NdsDto>> getNdsByRate(
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    ) {
        if (date == null) {
            date = ZonedDateTime.now();
        }
        ZonedDateTime finalDate = date;

        return responseHandler.executeOrThrow(() ->
                getSuccessResponse(ndsService.getBasicVatRate(finalDate, null, rate))
        );
    }

    @Override
    public ResultObj<List<NdsDto>> getNdsByCode(
            @RequestParam(name = "code", required = false) List<String> code,
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    ) {
        if (date == null) {
            date = ZonedDateTime.now();
        }
        

        
                getSuccessResponse(ndsService.getBasicVatRate(date, code, rate));

    }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
@Schema(description = "Информация о действующих ставках НДС на дату")
public class NdsDto implements Serializable {

    @Schema(
            description = "Ставка НДС",
            title = "Ставка НДС",
            example = "105",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String rate;

    @Schema(
            description = "Текстовое описание налога",
            title = "Описание налога",
            example = "5%/105%",
            type = "string",
            maxLength = 255,
            pattern = ".*$"
    )
    private String name;

    @Schema(
            description = "UUID код ставки НДС",
            title = "UUID ставки",
            example = "98971a55-7634-42b5-b1f8-ef31994eef54",
            type = "string",
            format = "uuid",
            maxLength = 36,
            pattern = "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"
    )
    private String id;

    @Schema(
            description = "Код налога",
            title = "Код налога",
            example = "01",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String code;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
@Schema(description = "Информация о действующих ставках НДС на дату (расширенная)")
public class NdsFullDto implements Serializable {

    @Schema(
            description = "Ставка НДС",
            title = "Ставка НДС",
            example = "105",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String rate;

    @Schema(
            description = "Текстовое описание налога",
            title = "Описание налога",
            example = "5%/105%",
            type = "string",
            maxLength = 255,
            pattern = ".*$"
    )
    private String name;

    @Schema(
            description = "UUID код ставки НДС",
            title = "UUID ставки",
            example = "98971a55-7634-42b5-b1f8-ef31994eef54",
            type = "string",
            format = "uuid",
            maxLength = 36,
            pattern = "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"
    )
    private String id;

    @Schema(
            description = "Тип ставки",
            title = "Тип ставки",
            example = "2",
            type = "integer",
            format = "int32"
    )
    private int rateType;

    @Schema(
            description = "Значение ставки (числитель)",
            title = "Числитель",
            example = "5",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String rateNominator;

    @Schema(
            description = "Значение расчётной ставки (знаменатель)",
            title = "Знаменатель",
            example = "105",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String rateDenominator;

    @Schema(
            description = "Дата начала срока действия налога",
            title = "Дата начала действия",
            example = "2018-12-31T21:00:00Z",
            type = "string",
            format = "date-time"
    )
    private ZonedDateTime rateDateStartZoned;

    @Schema(
            description = "Дата окончания срока действия налога",
            title = "Дата окончания действия",
            example = "2999-12-30T21:00:00Z",
            type = "string",
            format = "date-time"
    )
    private ZonedDateTime rateDateEndZoned;

    @Schema(
            description = "Код налога",
            title = "Код налога",
            example = "01",
            type = "string",
            maxLength = 255,
            pattern = "^\\d+$"
    )
    private String code;
}
```
