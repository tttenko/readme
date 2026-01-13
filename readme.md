```java
@Slf4j
@Validated
@RestController
@RequiredArgsConstructor
public class TerBanksControllerImpl implements TerBanksController {

    private final TerBankService terBankService;
    private final ResponseHandler responseHandler;

    @Override
    public ResultObj<List<TerBankDto>> getAllTerBanks() {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getAllTerBanks())
        );
    }

    @Override
    public ResultObj<List<TerBankDto>> getTerBank(
        @RequestParam(name = "tbCode")
        @NotEmpty(message = "Параметр tbCode должен содержать хотя бы одно значение")
        List<@NotBlank(message = "Код территориального банка не должен быть пустым") String> tbCodes
    ) {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getTerBanksByCodes(tbCodes))
        );
    }

    @Override
    public ResultObj<List<TerBankDto>> getTerBankById(
        @PathVariable("tbCode")
        @NotBlank(message = "tbCode не должен быть пустым")
        String tbCode
    ) {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getTerBanksByCodes(List.of(tbCode)))
        );
    }

    @Override
    public ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteALL() {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getAllTerBanksWithRequisite())
        );
    }

    @Override
    public ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisite(
        @RequestParam(name = "tbCode")
        @NotEmpty(message = "Параметр tbCode должен содержать хотя бы одно значение")
        List<@NotBlank(message = "Код территориального банка не должен быть пустым") String> tbCodes
    ) {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getTerBanksWithRequisiteByCodes(tbCodes))
        );
    }

    @Override
    public ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteById(
        @PathVariable("tbCode")
        @NotBlank(message = "tbCode не должен быть пустым")
        String tbCode
    ) {
        return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getTerBanksWithRequisiteByCodes(List.of(tbCode)))
        );
    }
}
```
