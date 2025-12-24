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
```
