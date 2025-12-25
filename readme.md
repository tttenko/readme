```java
public abstract class AbstractNdsController {

    protected final NdsService ndsService;
    protected final ResponseHandler responseHandler;

    protected AbstractNdsController(NdsService ndsService, ResponseHandler responseHandler) {
        this.ndsService = ndsService;
        this.responseHandler = responseHandler;
    }

    protected ResultObj<List<NdsDto>> getNdsByRateInternal(List<String> rate, ZonedDateTime date) {
        ZonedDateTime finalDate = (date == null) ? ZonedDateTime.now() : date;
        return responseHandler.executeOrThrow(() ->
                getSuccessResponse(ndsService.getBasicVatRate(finalDate, null, rate)));
    }

    protected ResultObj<List<NdsDto>> getNdsByCodeInternal(List<String> code, List<String> rate, ZonedDateTime date) {
        ZonedDateTime finalDate = (date == null) ? ZonedDateTime.now() : date;
        return getSuccessResponse(ndsService.getBasicVatRate(finalDate, code, rate));
    }
}

@RestController
@RequestMapping("/api/v1")
public class NdsControllerImpl extends AbstractNdsController implements NdsController {

    public NdsControllerImpl(NdsService ndsService, ResponseHandler responseHandler) {
        super(ndsService, responseHandler);
    }

    @GetMapping("/main-nds")
    public ResultObj<List<NdsDto>> getNdsByRate(List<String> rate, ZonedDateTime date) {
        return getNdsByRateInternal(rate, date);
    }

    @GetMapping("/main-nds-code")
    public ResultObj<List<NdsDto>> getNdsByCode(List<String> code, List<String> rate, ZonedDateTime date) {
        return getNdsByCodeInternal(code, rate, date);
    }
}

@RestController
@RequestMapping("/ui/v1")
public class NdsUiControllerImpl extends AbstractNdsController implements UiNdsController {

    public NdsUiControllerImpl(NdsService ndsService, ResponseHandler responseHandler) {
        super(ndsService, responseHandler);
    }

    @GetMapping("/main-nds")
    public ResultObj<List<NdsDto>> getNdsByRate(List<String> rate, ZonedDateTime date) {
        return getNdsByRateInternal(rate, date);
    }

    @GetMapping("/main-nds-code")
    public ResultObj<List<NdsDto>> getNdsByCode(List<String> code, List<String> rate, ZonedDateTime date) {
        return getNdsByCodeInternal(code, rate, date);
    }
}
```
