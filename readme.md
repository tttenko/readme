```java
package com.example.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.util.CollectionUtils;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

import static com.example.config.CacheConst.TB_ALL;
import static com.example.config.CacheConst.TB_BY_CODE;
import static com.example.config.CacheConst.TB_REQ_ALL;
import static com.example.config.CacheConst.TB_REQ_BY_CODE;

/**
 * Сервис ТерБанков с кешированием на Spring Cache (Caffeine).
 * Кеши:
 *  - tb_by_code: key = tbCode, value = TerBankDto
 *  - tb_all:     key = 'ALL',  value = List<TerBankDto>
 *  - tb_req_by_code: key = tbCode, value = TerBankWithRequisiteDto
 *  - tb_req_all:     key = 'ALL',  value = List<TerBankWithRequisiteDto>
 *
 * Бизнес-методы getTerBank/getTerBankRequisite остаются теми же по сигнатуре —
 * контроллер не меняем.
 */
@Slf4j
@Service
public class TerBankService extends AbstractMasterDataRequestService {

    private static final String TB_CODE = "tbCode";

    private final CacheManager cacheManager;
    private final TerBankMapper terBankMapper;
    private final TerBankWithRequisiteMapper terBankWithRequisiteMapper;

    public TerBankService(final SearchRequestProperties properties,
                          final HttpRequestHelper httpRequestHelper,
                          final ObjectMapper mapper,
                          final CacheManager cacheManager,
                          final TerBankMapper terBankMapper,
                          final TerBankWithRequisiteMapper terBankWithRequisiteMapper) {
        super(properties, httpRequestHelper, mapper);
        this.cacheManager = Objects.requireNonNull(cacheManager);
        this.terBankMapper = Objects.requireNonNull(terBankMapper);
        this.terBankWithRequisiteMapper = Objects.requireNonNull(terBankWithRequisiteMapper);
    }

    // -------- Публичный API, который дергает контроллер --------

    /** Список ТБ по кодам (или все, если список пуст). */
    @Nonnull
    public ResultObj<List<TerBankDto>> getTerBank(@Nullable final List<String> tbCodes) {
        final List<TerBankDto> data = CollectionUtils.isEmpty(tbCodes)
                ? getAllBanks()                                  // кэш tb_all
                : getBanksByCodesBatch(normalize(tbCodes));      // кэш tb_by_code + дозагрузка
        return getSuccessResponse(data);
    }

    /** Список ТБ с реквизитами по кодам (или все, если список пуст). */
    @Nonnull
    public ResultObj<List<TerBankWithRequisiteDto>> getTerBankRequisite(@Nullable final List<String> tbCodes) {
        final List<TerBankWithRequisiteDto> data = CollectionUtils.isEmpty(tbCodes)
                ? getAllBanksWithRequisite()                         // кэш tb_req_all
                : getBanksRequisiteByCodesBatch(normalize(tbCodes)); // кэш tb_req_by_code + дозагрузка
        return getSuccessResponse(data);
    }

    // -------- Кэширующие методы --------

    /** Кэш «все ТБ». */
    @Cacheable(cacheNames = TB_ALL, key = "'ALL'")
    @Nonnull
    public List<TerBankDto> getAllBanks() {
        // В старом коде "allLoaded" делалось через null/empty. Здесь явно грузим все.
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                null, // согласно прежней семантике: null -> «все»
                SearchRequestProperties.Context.BOOK
        );
        return createResult(response, terBankMapper);
    }

    /** Кэш «все ТБ с реквизитами». */
    @Cacheable(cacheNames = TB_REQ_ALL, key = "'ALL'")
    @Nonnull
    public List<TerBankWithRequisiteDto> getAllBanksWithRequisite() {
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                null,
                SearchRequestProperties.Context.BOOK
        );
        return createWithAttribute(response, terBankWithRequisiteMapper);
    }

    /** Кэш по одному коду (ТБ). */
    @Cacheable(cacheNames = TB_BY_CODE, key = "#code")
    @Nullable
    public TerBankDto getByCode(@Nonnull final String code) {
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                List.of(code),
                SearchRequestProperties.Context.BOOK
        );
        final List<TerBankDto> list = createResult(response, terBankMapper);
        return list.isEmpty() ? null : list.get(0);
    }

    /** Кэш по одному коду (ТБ с реквизитами). */
    @Cacheable(cacheNames = TB_REQ_BY_CODE, key = "#code")
    @Nullable
    public TerBankWithRequisiteDto getRequisiteByCode(@Nonnull final String code) {
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                List.of(code),
                SearchRequestProperties.Context.BOOK
        );
        final List<TerBankWithRequisiteDto> list = createWithAttribute(response, terBankWithRequisiteMapper);
        return list.isEmpty() ? null : list.get(0);
    }

    // -------- Batch-aware дозагрузка через CacheManager --------

    /** Достаёт из кеша всё, чего хватает, недостающее грузит одним запросом и кладёт в кеш. */
    @Nonnull
    private List<TerBankDto> getBanksByCodesBatch(@Nonnull final List<String> codes) {
        final Cache cache = cacheManager.getCache(TB_BY_CODE);
        final Map<String, TerBankDto> hits = new LinkedHashMap<>();
        final List<String> miss = new ArrayList<>();
        for (String code : codes) {
            final TerBankDto cached = (cache != null) ? cache.get(code, TerBankDto.class) : null;
            if (cached != null) {
                hits.put(code, cached);
            } else {
                miss.add(code);
            }
        }

        final List<TerBankDto> loaded = miss.isEmpty() ? List.of() : loadBanksByCodes(miss);
        if (cache != null) {
            for (TerBankDto dto : loaded) {
                final String code = extractTbCode(dto);
                if (code != null) cache.put(code, dto);
            }
        }

        // Собираем результат в исходном порядке входных кодов
        final Map<String, TerBankDto> byCode = new HashMap<>();
        hits.forEach((code, dto) -> byCode.put(code, dto));
        for (TerBankDto dto : loaded) {
            final String code = extractTbCode(dto);
            if (code != null) byCode.put(code, dto);
        }
        return codes.stream()
                .map(byCode::get)
                .filter(Objects::nonNull)
                .toList();
    }

    /** Аналогично batch для реквизитов. */
    @Nonnull
    private List<TerBankWithRequisiteDto> getBanksRequisiteByCodesBatch(@Nonnull final List<String> codes) {
        final Cache cache = cacheManager.getCache(TB_REQ_BY_CODE);
        final Map<String, TerBankWithRequisiteDto> hits = new LinkedHashMap<>();
        final List<String> miss = new ArrayList<>();
        for (String code : codes) {
            final TerBankWithRequisiteDto cached = (cache != null) ? cache.get(code, TerBankWithRequisiteDto.class) : null;
            if (cached != null) {
                hits.put(code, cached);
            } else {
                miss.add(code);
            }
        }

        final List<TerBankWithRequisiteDto> loaded = miss.isEmpty() ? List.of() : loadBanksRequisiteByCodes(miss);
        if (cache != null) {
            for (TerBankWithRequisiteDto dto : loaded) {
                final String code = extractTbCode(dto);
                if (code != null) cache.put(code, dto);
            }
        }

        final Map<String, TerBankWithRequisiteDto> byCode = new HashMap<>();
        hits.forEach((code, dto) -> byCode.put(code, dto));
        for (TerBankWithRequisiteDto dto : loaded) {
            final String code = extractTbCode(dto);
            if (code != null) byCode.put(code, dto);
        }
        return codes.stream()
                .map(byCode::get)
                .filter(Objects::nonNull)
                .toList();
    }

    // -------- Загрузка из внешнего источника (одним запросом) --------

    @Nonnull
    private List<TerBankDto> loadBanksByCodes(@Nonnull final List<String> codes) {
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                codes,
                SearchRequestProperties.Context.BOOK
        );
        return createResult(response, terBankMapper);
    }

    @Nonnull
    private List<TerBankWithRequisiteDto> loadBanksRequisiteByCodes(@Nonnull final List<String> codes) {
        final GetItemsSearchResponse response = requestDataWithAttribute(
                properties.getSlugValueForTerBank(),
                codes,
                SearchRequestProperties.Context.BOOK
        );
        return createWithAttribute(response, terBankWithRequisiteMapper);
    }

    // -------- Утилиты --------

    /** Нормализуем вход (trim, distinct, не-null/не-пустые). */
    @Nonnull
    private static List<String> normalize(@Nonnull final List<String> codes) {
        return codes.stream()
                .filter(Objects::nonNull)
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .distinct()
                .collect(Collectors.toCollection(ArrayList::new));
    }

    /** Извлечь tbCode из DTO. Подстрой при необходимости под фактическое поле. */
    @Nullable
    private static String extractTbCode(@Nonnull final TerBankDto dto) {
        return dto.getTbCode(); // <— предполагается геттер tbCode
    }

    @Nullable
    private static String extractTbCode(@Nonnull final TerBankWithRequisiteDto dto) {
        return dto.getTbCode(); // <— предполагается геттер tbCode
    }
}

public static final String TB_BY_CODE   = "tb_by_code";
public static final String TB_ALL       = "tb_all";
public static final String TB_REQ_BY_CODE = "tb_req_by_code";
public static final String TB_REQ_ALL     = "tb_req_all";

public static final List<String> NAMES = List.of(
    TB_BY_CODE, TB_ALL, TB_REQ_BY_CODE, TB_REQ_ALL
);

----------------------------------------------------------------------------------ОБЪЯСНЕНИЕ---------------------------------------------------------------------------------------------------
Ниже — подробное разъяснение того, как работает TerBankService в новой реализации с Spring Cache (Caffeine): что именно кэшируется, как сервис ходит в кэш, зачем батчи, и за что отвечает каждый метод.

1) Что кэшируем и какими ключами

Мы используем 4 кэша (каждый — отдельный cacheName у Spring Cache):

Кэш	Назначение	Ключ (key)	Значение (value)
tb_by_code	Один банк по коду	tbCode	TerBankDto
tb_all	Все банки	'ALL'	List<TerBankDto>
tb_req_by_code	Один банк с реквизитами	tbCode	TerBankWithRequisiteDto
tb_req_all	Все банки с реквизитами	'ALL'	List<TerBankWithRequisiteDto>

TTL/размер задаются в CacheConfig через Caffeine.newBuilder() (например, expireAfterWrite(10m), maximumSize(10_000), recordStats()).

2) Общая идея взаимодействия с кэшем

Для одного кода — используем метод с @Cacheable: при первом запросе делает сетевой вызов и кладёт результат в кэш; при повторном — отдаёт из памяти.

Для списка кодов — работаем батчем:

Сначала вытаскиваем из кэша всё, что уже есть (хиты).

Собираем только промахи (отсутствующие коды).

Если промахи есть — одним сетевым запросом грузим недостающее и размазываем в кэш по ключам.

Возвращаем объединённый результат в порядке входных кодов.

Для «все банки» — отдельные методы с @Cacheable на ключ "'ALL'". Они не связаны с по-кодовыми кэшами: это осознанно (чёткая изоляция кэш-линий).

3) Почему используем и @Cacheable, и программное API CacheManager
Зачем @Cacheable

Методы:

getByCode(String) → @Cacheable(cacheNames = TB_BY_CODE, key = "#code")

getRequisiteByCode(String) → @Cacheable(cacheNames = TB_REQ_BY_CODE, key = "#code")

getAllBanks() → @Cacheable(cacheNames = TB_ALL, key = "'ALL'")

getAllBanksWithRequisite() → @Cacheable(cacheNames = TB_REQ_ALL, key = "'ALL'")

Аннотации дают «ленивую» загрузку и простую инвалидацию по TTL, без ручного кода.

Зачем программный доступ через CacheManager

Внутри класса вызов this.getByCode(code) не задействует AOP‑прокси (@Cacheable не сработает, это «self‑invocation»). А нам нужно:

одним запросом догружать несколько кодов (минимум сетевых вызовов),

руками положить результаты в кэш для всех недостающих ключей.

Поэтому в батч‑методах мы работаем так:

Cache cache = cacheManager.getCache(TB_BY_CODE);
TerBankDto hit = cache.get(code, TerBankDto.class) // достать из кэша
...
cache.put(code, dto); // положить в кэш результаты массовой загрузки

4) Разбор методов (кто за что отвечает)
Публичные методы (вызывает контроллер)

ResultObj<List<TerBankDto>> getTerBank(List<String> tbCodes)

Если список пуст → getAllBanks() (кэш tb_all).

Если список не пуст → getBanksByCodesBatch(normalize(tbCodes)) (батч‑логика + кэш tb_by_code).

Оборачивает в ResultObj (как у тебя делалось в контроллере через handleBankRequest).

ResultObj<List<TerBankWithRequisiteDto>> getTerBankRequisite(List<String> tbCodes)

Аналогично, но с реквизитами и кэшами tb_req_*.

Эти методы сами не кэшируются: они агрегируют результаты кэширующих методов и/или батч‑дозагрузку.

Кэширующие методы «одиночки» и «все»

@Cacheable TB_ALL → List<TerBankDto> getAllBanks()
Делает «полный» запрос к МД и кладёт результат в tb_all под ключом ALL.

@Cacheable TB_REQ_ALL → List<TerBankWithRequisiteDto> getAllBanksWithRequisite()

@Cacheable TB_BY_CODE → TerBankDto getByCode(String code)

@Cacheable TB_REQ_BY_CODE → TerBankWithRequisiteDto getRequisiteByCode(String code)

Плюс: можно включить sync = true на одиночных методах, чтобы под одним ключом одновременно грузил один поток:

@Cacheable(cacheNames = TB_BY_CODE, key = "#code", sync = true)

Батч‑методы (приватные; оптимизируют сеть)

List<TerBankDto> getBanksByCodesBatch(List<String> codes)

Достаёт хиты из tb_by_code.

Собирает промахи в miss.

loadBanksByCodes(miss) — один сетевой вызов.

Результаты кладёт в tb_by_code (cache.put(code, dto)).

Склеивает хиты и загруженное, возвращает в исходном порядке входа.

List<TerBankWithRequisiteDto> getBanksRequisiteByCodesBatch(List<String> codes)
Точно так же, но для tb_req_by_code.

Загрузчики (приватные; только сеть → DTO)

loadBanksByCodes(List<String> codes)
Вызывает твой requestDataWithAttribute(..., codes, ...) и маппер terBankMapper.

loadBanksRequisiteByCodes(List<String> codes)
То же для реквизитов и terBankWithRequisiteMapper.

Утилиты (приватные; чистота кода)

normalize(List<String>) — фильтрует null/"", делает distinct() с сохранением порядка.
→ гарантирует стабильные и детерминированные ключи кэша.

extractTbCode(dto) — единое место, где тянем tbCode из DTO (если поле переименуют — правим тут, а не по проекту).

Почему много приватных методов?
SOLID/читаемость/переиспользование:

Чётко отделены задачи: «достать из кэша», «сходить в сеть», «положить в кэш», «нормализовать вход».

Легко точечно тестировать (через публичные), расширять (tb_by_inn/tb_by_bic) и менять мапперы/загрузку, не трогая кэш‑логику.

5) Жизненный цикл запроса — пошагово
/tb?tbCodes=044525225,0123456789

Контроллер → getTerBank(codes).

normalize() → ["044525225","0123456789"].

getBanksByCodesBatch():

cache.get("044525225") → промах (первый раз).

cache.get("0123456789") → промах.

miss = ["044525225","0123456789"] → ОДИН вызов loadBanksByCodes(miss).

Разложили put(code, dto) для каждого результата → теперь в кэше два хита.

Вернули [dto1, dto2] (в порядке входа).

Повторный запрос этими же кодами — оба хита, сети нет.

/tb (без параметров)

Контроллер → getTerBank(empty).

Вызывает getAllBanks() (кэш tb_all, ключ ALL).

Первый раз — сетевой вызов → положили в кэш; дальше — хиты.

6) Важные нюансы

Null‑значения. По умолчанию Spring Cache не кэширует null. В наших одиночных методах, если банк не найден, вернётся null и он не будет кэширован — повторные запросы к «плохому» коду снова пойдут в сеть. Если хотите кэшировать «не найдено», включите CaffeineCacheManager.setAllowNullValues(true) или храните «пустой» DTO/маркер, и добавьте unless в @Cacheable при необходимости.

Согласованность tb_all и tb_by_code. Кэши независимы. Загрузка «всех» не прогревает по‑кодовые ключи (и наоборот). Это упрощает логику и исключает «сюрпризы» при инвалидации. Если нужен автопрогрев по кодам после getAllBanks() — можно в конце метода пройтись по списку и сделать cache.put(dto.getTbCode(), dto).

Параллельные запросы к одному коду. Добавьте sync = true к @Cacheable одиночек, чтобы несколько потоков не дублировали загрузку одного и того же ключа.

Метрики. Включён recordStats(). При желании можно чтить статистику Caffeine:

var nativeCache = (com.github.benmanes.caffeine.cache.Cache<?, ?>)
    ((org.springframework.cache.caffeine.CaffeineCache) cacheManager.getCache(TB_BY_CODE)).getNativeCache();
log.info("hitRate={}", nativeCache.stats().hitRate());


Инвалидация. Основная — TTL. Плюс у тебя есть CacheEvictScheduler на cron, который очищает все кэши через CacheManager (админ. операция/интеграционные тесты).

7) Почему батчи — это важно

Без батча сервис при списке из N кодов сделал бы N сетевых запросов (по одному на код). Батч‑подход:

уменьшает сетевые RTT до 1 запроса на все промахи;

снижает нагрузку на внешний сервис МД;

поддерживает порядок в ответе, соответствующий входным кодам;

не ломает семантику кэширования — хиты забираем из памяти, промахи догружаем и кладём в кэш для будущих обращений.

8) Краткий «чит‑лист» по методам

getTerBank(...) / getTerBankRequisite(...) — оркестраторы (выбор ветки all/batch, формирование ResultObj).

getAllBanks() / getAllBanksWithRequisite() — @Cacheable «все» (ключ ALL).

getByCode(code) / getRequisiteByCode(code) — @Cacheable одиночные (ключ code).

getBanksByCodesBatch(codes) / getBanksRequisiteByCodesBatch(codes) — батч‑дозагрузка недостающих ключей + ручное cache.put.

loadBanksByCodes(...) / loadBanksRequisiteByCodes(...) — один сетевой запрос к МД + маппинг.

normalize(...), extractTbCode(...) — служебные; держат ключи детерминированными и код аккуратным.

Если захочешь — покажу, как дополнительно «нагреть» tb_by_code внутри getAllBanks() и добавлю sync=true + unit‑тесты на батчи/хиты/промахи.
```
