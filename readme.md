```java
/**
 * HTTP-interface для вызова MasterData API через WebClient/HttpServiceProxyFactory.
 */
@HttpExchange(url = "/api/v1", accept = MediaType.APPLICATION_JSON_VALUE)
public interface MasterDataClientApi {

    // -------------------- AdapterController (/api/v1/info) --------------------

    @GetExchange("/info/uom/all")
    ResultObj<List<UomBankDto>> getAllMeasures();

    @GetExchange("/info/uom")
    ResultObj<List<UomBankDto>> getMeasuresByCodes(
            @RequestParam(required = false, name = "uomCode") List<String> uomCode
    );

    @GetExchange("/info/uom/{uomCode}")
    ResultObj<List<UomBankDto>> getMeasuresById(
            @PathVariable(name = "uomCode") String uomCode
    );

    @GetExchange("/info/material")
    ResultObj<List<MaterialDto>> getMaterialByCodes(
            @RequestParam(required = false, name = "materialCode") List<String> materialCodes
    );

    @GetExchange("/info/material/{materialCode}")
    ResultObj<List<MaterialDto>> getMaterialById(
            @PathVariable(name = "materialCode") String materialCode
    );

    @GetExchange("/info/material-type/all")
    ResultObj<List<MaterialTypeDto>> getAllMaterialTypes();

    @GetExchange("/info/material-type")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeIds(
            @RequestParam(name = "typeId", required = false) List<String> typeIds
    );

    @GetExchange("/info/material-type/{typeId}")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeById(
            @PathVariable(name = "typeId") String typeId
    );

    // -------------------- CacheController (/api/v1/cache) --------------------

    @GetExchange("/cache/invalidate")
    ResultObj<Object> invalidate();

    @GetExchange("/cache/status")
    ResultObj<CacheStatusResponse> getStatus();

    // -------------------- CountryController (/api/v1/info/country) --------------------

    @GetExchange("/info/country")
    ResultObj<List<CountryDto>> searchCountryByCodes(
            @RequestParam(name = "countryCode") List<String> countryCodes
    );

    @GetExchange("/info/country/{countryCode}")
    ResultObj<List<CountryDto>> searchCountryByCode(
            @PathVariable("countryCode") String countryCode
    );

    @GetExchange("/info/country/all")
    ResultObj<List<CountryDto>> getAllCountries();

    // -------------------- CurrencyController (/api/v1/info/currency) --------------------

    @GetExchange("/info/currency/all")
    ResultObj<List<CurrencyDto>> getAllCurrencies();

    @GetExchange("/info/currency")
    ResultObj<List<CurrencyDto>> searchCurrenciesByCodes(
            @RequestParam(required = false, name = "currencyCode") List<String> currencyCodes
    );

    @GetExchange("/info/currency/{currencyCode}")
    ResultObj<List<CurrencyDto>> searchCurrencyByCode(
            @PathVariable("currencyCode") String currencyCode
    );

    // -------------------- NdsController (/api/v1) --------------------

    @GetExchange("/main-nds")
    ResultObj<List<NdsDto>> getNdsByRate(
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    );

    @GetExchange("/main-nds-code")
    ResultObj<List<NdsDto>> getNdsByCode(
            @RequestParam(name = "code", required = false) List<String> code,
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    );

    // -------------------- RegionController (/api/v1/info) --------------------

    @GetExchange("/info/region_code")
    ResultObj<List<RegionDto>> searchRegionsByCode(
            @RequestParam(name = "regionCode") List<String> regionCodes
    );

    @GetExchange("/info/{regionCode}")
    ResultObj<List<RegionDto>> searchRegionByCode(
            @PathVariable("regionCode") String regionCode
    );

    @GetExchange("/info/region_code/all")
    ResultObj<List<RegionDto>> getAllRegions();

    // -------------------- SupplierController (/api/v1) --------------------

    @GetExchange("/supplier-search")
    ResultObj<List<CounterpartyDto>> searchSupplier(
            @RequestParam(name = "inn") String inn,
            @RequestParam(required = false, name = "kpp") String kpp
    );

    @GetExchange("/supplier")
    ResultObj<List<CounterpartyDto>> getSupplier(
            @RequestParam(required = false, name = "id") List<String> listOfId
    );

    @GetExchange("/supplier/{id}")
    ResultObj<List<CounterpartyDto>> getSupplierById(
            @PathVariable("id") String id
    );

    @GetExchange("/supplier-bank-requisite")
    ResultObj<List<BankDto>> getSupplierRequisite(
            @RequestParam(name = "id") List<String> listOfId
    );

    @GetExchange("/supplier-bank-requisite/{id}")
    ResultObj<List<BankDto>> getSupplierRequisiteById(
            @PathVariable("id") String id
    );

    // -------------------- TerBanksController (/api/v1/info) --------------------

    @GetExchange("/info/tb/all")
    ResultObj<List<TerBankDto>> getAllTerBanks();

    @GetExchange("/info/tb")
    ResultObj<List<TerBankDto>> getTerBank(
            @RequestParam(name = "tbCode", required = false) List<String> tbCode
    );

    @GetExchange("/info/tb/{tbCode}")
    ResultObj<List<TerBankDto>> getTerBankById(
            @PathVariable("tbCode") String tbCode
    );

    @GetExchange("/info/tb-requisite/all")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteALL();

    @GetExchange("/info/tb-requisite")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisite(
            @RequestParam(name = "tbCode", required = false) List<String> tbCode
    );

    @GetExchange("/info/tb-requisite/{tbCode}")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteById(
            @PathVariable("tbCode") String tbCode
    );
}


/**
 * HTTP-interface для вызова MasterData API через WebClient/HttpServiceProxyFactory.
 */
@HttpExchange(url = "/api/v1", accept = MediaType.APPLICATION_JSON_VALUE)
public interface MasterDataClientApi {

    // -------------------- AdapterController (/api/v1/info) --------------------

    @GetExchange("/info/uom/all")
    ResultObj<List<UomBankDto>> getAllMeasures();

    @GetExchange("/info/uom")
    ResultObj<List<UomBankDto>> getMeasuresByCodes(
            @RequestParam(required = false, name = "uomCode") List<String> uomCode
    );

    @GetExchange("/info/uom/{uomCode}")
    ResultObj<List<UomBankDto>> getMeasuresById(
            @PathVariable(name = "uomCode") String uomCode
    );

    @GetExchange("/info/material")
    ResultObj<List<MaterialDto>> getMaterialByCodes(
            @RequestParam(required = false, name = "materialCode") List<String> materialCodes
    );

    @GetExchange("/info/material/{materialCode}")
    ResultObj<List<MaterialDto>> getMaterialById(
            @PathVariable(name = "materialCode") String materialCode
    );

    @GetExchange("/info/material-type/all")
    ResultObj<List<MaterialTypeDto>> getAllMaterialTypes();

    @GetExchange("/info/material-type")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeIds(
            @RequestParam(name = "typeId", required = false) List<String> typeIds
    );

    @GetExchange("/info/material-type/{typeId}")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeById(
            @PathVariable(name = "typeId") String typeId
    );

    // -------------------- CacheController (/api/v1/cache) --------------------

    @GetExchange("/cache/invalidate")
    ResultObj<Object> invalidate();

    @GetExchange("/cache/status")
    ResultObj<CacheStatusResponse> getStatus();

    // -------------------- CountryController (/api/v1/info/country) --------------------

    @GetExchange("/info/country")
    ResultObj<List<CountryDto>> searchCountryByCodes(
            @RequestParam(name = "countryCode") List<String> countryCodes
    );

    @GetExchange("/info/country/{countryCode}")
    ResultObj<List<CountryDto>> searchCountryByCode(
            @PathVariable("countryCode") String countryCode
    );

    @GetExchange("/info/country/all")
    ResultObj<List<CountryDto>> getAllCountries();

    // -------------------- CurrencyController (/api/v1/info/currency) --------------------

    @GetExchange("/info/currency/all")
    ResultObj<List<CurrencyDto>> getAllCurrencies();

    @GetExchange("/info/currency")
    ResultObj<List<CurrencyDto>> searchCurrenciesByCodes(
            @RequestParam(required = false, name = "currencyCode") List<String> currencyCodes
    );

    @GetExchange("/info/currency/{currencyCode}")
    ResultObj<List<CurrencyDto>> searchCurrencyByCode(
            @PathVariable("currencyCode") String currencyCode
    );

    // -------------------- NdsController (/api/v1) --------------------

    @GetExchange("/main-nds")
    ResultObj<List<NdsDto>> getNdsByRate(
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    );

    @GetExchange("/main-nds-code")
    ResultObj<List<NdsDto>> getNdsByCode(
            @RequestParam(name = "code", required = false) List<String> code,
            @RequestParam(name = "rate", required = false) List<String> rate,
            @RequestParam(name = "date", required = false)
            @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) ZonedDateTime date
    );

    // -------------------- RegionController (/api/v1/info) --------------------

    @GetExchange("/info/region_code")
    ResultObj<List<RegionDto>> searchRegionsByCode(
            @RequestParam(name = "regionCode") List<String> regionCodes
    );

    @GetExchange("/info/{regionCode}")
    ResultObj<List<RegionDto>> searchRegionByCode(
            @PathVariable("regionCode") String regionCode
    );

    @GetExchange("/info/region_code/all")
    ResultObj<List<RegionDto>> getAllRegions();

    // -------------------- SupplierController (/api/v1) --------------------

    @GetExchange("/supplier-search")
    ResultObj<List<CounterpartyDto>> searchSupplier(
            @RequestParam(name = "inn") String inn,
            @RequestParam(required = false, name = "kpp") String kpp
    );

    @GetExchange("/supplier")
    ResultObj<List<CounterpartyDto>> getSupplier(
            @RequestParam(required = false, name = "id") List<String> listOfId
    );

    @GetExchange("/supplier/{id}")
    ResultObj<List<CounterpartyDto>> getSupplierById(
            @PathVariable("id") String id
    );

    @GetExchange("/supplier-bank-requisite")
    ResultObj<List<BankDto>> getSupplierRequisite(
            @RequestParam(name = "id") List<String> listOfId
    );

    @GetExchange("/supplier-bank-requisite/{id}")
    ResultObj<List<BankDto>> getSupplierRequisiteById(
            @PathVariable("id") String id
    );

    // -------------------- TerBanksController (/api/v1/info) --------------------

    @GetExchange("/info/tb/all")
    ResultObj<List<TerBankDto>> getAllTerBanks();

    @GetExchange("/info/tb")
    ResultObj<List<TerBankDto>> getTerBank(
            @RequestParam(name = "tbCode", required = false) List<String> tbCode
    );

    @GetExchange("/info/tb/{tbCode}")
    ResultObj<List<TerBankDto>> getTerBankById(
            @PathVariable("tbCode") String tbCode
    );

    @GetExchange("/info/tb-requisite/all")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteALL();

    @GetExchange("/info/tb-requisite")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisite(
            @RequestParam(name = "tbCode", required = false) List<String> tbCode
    );

    @GetExchange("/info/tb-requisite/{tbCode}")
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteById(
            @PathVariable("tbCode") String tbCode
    );
}

dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
  </dependency>

  <!-- @Valid + @NotBlank/@NotEmpty/... -->
  <dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
  </dependency>

  <!-- OpenAPI аннотации: @Operation/@ApiResponses/@Tag/@Parameter/... -->
  <dependency>
    <groupId>io.swagger.core.v3</groupId>
    <artifactId>swagger-annotations-jakarta</artifactId>
  </dependency>

  <!-- Если DTO размечены Jackson-аннотациями -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
  </dependency>

  <!-- Твой общий wrapper ResultObj -->
  <dependency>
    <groupId>ru.sber.cs.core</groupId>
    <artifactId>cs-core-rest-response</artifactId>
  </dependency>

  <!-- Lombok (лучше provided, чтобы не тащить транзитивно) -->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <scope>provided</scope>
    <optional>true</optional>
  </dependency>
```
