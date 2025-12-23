```java
@Slf4j
@Component
@RequiredArgsConstructor
public class SmokeRunner implements CommandLineRunner {

    private final MasterDataClientApi masterDataClientApi;

    private final ObjectMapper om = new ObjectMapper().findAndRegisterModules();

    @Override
    public void run(String... args) {
        // ---------------- AdapterController ----------------
        var uoms = safeList("Adapter:getAllMeasures", masterDataClientApi::getAllMeasures);
        var uomCode = pickFirst(uoms, "uomCode", "code", "id");
        if (uomCode != null) {
            safeList("Adapter:getMeasuresByCodes", () -> masterDataClientApi.getMeasuresByCodes(List.of(uomCode)));
            safeList("Adapter:getMeasuresById", () -> masterDataClientApi.getMeasuresById(uomCode));
        }

        // material (в контроллере параметр optional, можно null)
        var materials = safeList("Adapter:getMaterialByCodes(null)", () -> masterDataClientApi.getMaterialByCodes(null));
        var materialCode = pickFirst(materials, "materialCode", "code", "id");
        if (materialCode != null) {
            safeList("Adapter:getMaterialById", () -> masterDataClientApi.getMaterialById(materialCode));
        }

        var materialTypes = safeList("Adapter:getAllMaterialTypes", masterDataClientApi::getAllMaterialTypes);
        var typeId = pickFirst(materialTypes, "typeId", "id", "code");
        if (typeId != null) {
            safeList("Adapter:getMaterialTypeIds", () -> masterDataClientApi.getMaterialTypeIds(List.of(typeId)));
            safeList("Adapter:getMaterialTypeById", () -> masterDataClientApi.getMaterialTypeById(typeId));
        }

        // ---------------- CacheController ----------------
        safeObj("Cache:invalidate", masterDataClientApi::invalidate);
        safeObj("Cache:getStatus", masterDataClientApi::getStatus);

        // ---------------- CountryController ----------------
        var countries = safeList("Country:getAllCountries", masterDataClientApi::getAllCountries);
        var countryCode = pickFirst(countries, "alpha2", "countryCode", "code", "id");
        if (countryCode != null) {
            safeList("Country:searchCountryByCodes", () -> masterDataClientApi.searchCountryByCodes(List.of(countryCode)));
            safeList("Country:searchCountryByCode", () -> masterDataClientApi.searchCountryByCode(countryCode));
        }

        // ---------------- CurrencyController ----------------
        var currencies = safeList("Currency:getAllCurrencies", masterDataClientApi::getAllCurrencies);
        var currencyCode = pickFirst(currencies, "currencyCode", "code", "id");
        if (currencyCode != null) {
            safeList("Currency:searchCurrenciesByCodes", () -> masterDataClientApi.searchCurrenciesByCodes(List.of(currencyCode)));
            safeList("Currency:searchCurrencyByCode", () -> masterDataClientApi.searchCurrencyByCode(currencyCode));
        }

        // ---------------- NdsController ----------------
        safeList("Nds:getNdsByRate(null,null)", () -> masterDataClientApi.getNdsByRate(null, null));
        safeList("Nds:getNdsByCode(null,null,null)", () -> masterDataClientApi.getNdsByCode(null, null, null));

        // ---------------- RegionController ----------------
        var regions = safeList("Region:getAllRegions", masterDataClientApi::getAllRegions);
        var regionCode = pickFirst(regions, "regionCode", "code", "id");
        if (regionCode != null) {
            safeList("Region:searchRegionsByCode", () -> masterDataClientApi.searchRegionsByCode(List.of(regionCode)));
            safeList("Region:searchRegionByCode", () -> masterDataClientApi.searchRegionByCode(regionCode));
        }

        // ---------------- SupplierController ----------------
        // В javadoc у тебя: если список id пустой — вернет всех, поэтому дергаем (null)
        var suppliers = safeList("Supplier:getSupplier(null)", () -> masterDataClientApi.getSupplier(null));
        var supplierId = pickFirst(suppliers, "id", "supplierId", "counterpartyId", "uuid");
        var inn = pickFirst(suppliers, "inn");
        var kpp = pickFirst(suppliers, "kpp");

        if (inn != null) {
            safeList("Supplier:searchSupplier(inn,kpp)", () -> masterDataClientApi.searchSupplier(inn, kpp));
        }
        if (supplierId != null) {
            safeList("Supplier:getSupplierById", () -> masterDataClientApi.getSupplierById(supplierId));
            safeList("Supplier:getSupplierRequisite", () -> masterDataClientApi.getSupplierRequisite(List.of(supplierId)));
            safeList("Supplier:getSupplierRequisiteById", () -> masterDataClientApi.getSupplierRequisiteById(supplierId));
        }

        // ---------------- TerBanksController ----------------
        var tbs = safeList("TerBanks:getAllTerBanks", masterDataClientApi::getAllTerBanks);
        var tbCode = pickFirst(tbs, "tbCode", "code", "id");
        safeList("TerBanks:getTerBank(tbCode?)",
                () -> masterDataClientApi.getTerBank(tbCode == null ? null : List.of(tbCode)));
        if (tbCode != null) {
            safeList("TerBanks:getTerBankById", () -> masterDataClientApi.getTerBankById(tbCode));
        }

        var tbsReq = safeList("TerBanks:getTerBanksRequisiteALL", masterDataClientApi::getTerBanksRequisiteALL);
        var tbReqCode = pickFirst(tbsReq, "tbCode", "code", "id");
        if (tbReqCode == null) tbReqCode = tbCode;

        safeList("TerBanks:getTerBanksRequisite(tbCode?)",
                () -> masterDataClientApi.getTerBanksRequisite(tbReqCode == null ? null : List.of(tbReqCode)));
        if (tbReqCode != null) {
            safeList("TerBanks:getTerBanksRequisiteById", () -> masterDataClientApi.getTerBanksRequisiteById(tbReqCode));
        }

        log.info("✅ Smoke finished");
    }

    // -------- helpers --------

    private <T> List<T> safeList(String name, Supplier<ResultObj<List<T>>> call) {
        try {
            var res = call.get();
            var data = (res != null ? res.getData() : null);
            int size = (data != null ? data.size() : 0);
            log.info("{} -> size={}", name, size);
            return data != null ? data : List.of();
        } catch (Exception e) {
            log.warn("{} -> FAILED: {}", name, e.toString());
            return List.of();
        }
    }

    private <T> T safeObj(String name, Supplier<ResultObj<T>> call) {
        try {
            var res = call.get();
            var data = (res != null ? res.getData() : null);
            // чтобы увидеть в логах "что пришло", но без огромных простыней
            log.info("{} -> {}", name, data == null ? "null" : toJsonShort(data));
            return data;
        } catch (Exception e) {
            log.warn("{} -> FAILED: {}", name, e.toString());
            return null;
        }
    }

    private String pickFirst(List<?> list, String... keys) {
        if (list == null || list.isEmpty()) return null;
        return pickField(list.get(0), keys);
    }

    private String pickField(Object dto, String... keys) {
        if (dto == null) return null;
        Map<String, Object> m = om.convertValue(dto, new TypeReference<Map<String, Object>>() {});
        for (String k : keys) {
            Object v = m.get(k);
            if (v == null) continue;
            String s = v.toString();
            if (!s.isBlank()) return s;
        }
        return null;
    }

    private String toJsonShort(Object o) {
        try {
            String json = om.writeValueAsString(o);
            return json.length() > 400 ? json.substring(0, 400) + "...(truncated)" : json;
        } catch (Exception e) {
            return o.toString();
        }
    }
}
```
