```java

package ru.sber.cs.supplier.portal.masterdata.services.impl.currency;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.MockedStatic;
import org.mockito.Mockito;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.cache.annotation.EnableCaching;

import ru.sber.cs.supplier.portal.masterdata.models.md.GetItemsSearchResponse;
import ru.sber.cs.supplier.portal.masterdata.models.adapter.CurrencyDto;
import ru.sber.cs.supplier.portal.masterdata.props.SearchRequestProperties;
import ru.sber.cs.supplier.portal.masterdata.services.impl.BaseMasterDataRequestService;
import ru.sber.cs.supplier.portal.masterdata.services.impl.currency.mapper.CurrencyMapper;
import ru.sber.cs.supplier.portal.masterdata.utils.CacheIntrospection;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;
import static org.mockito.Mockito.CALLS_REAL_METHODS;

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = CurrencyCacheOpsTest.Config.class)
@Import(CurrencyCacheOps.class)
class CurrencyCacheOpsTest {

  @TestConfiguration
  @EnableCaching(proxyTargetClass = true)
  static class Config {
    @Bean
    CacheManager cacheManager() {
      var mgr = new CaffeineCacheManager(CurrencyService2.CURRENCY_ALL);
      mgr.setCaffeine(
          Caffeine.newBuilder()
              .recordStats()
              .maximumSize(1_000)
      );
      return mgr;
    }
  }

  @org.springframework.beans.factory.annotation.Autowired
  private CurrencyCacheOps currencyCacheOps;

  @org.springframework.beans.factory.annotation.Autowired
  private CacheManager cacheManager;

  @org.springframework.boot.test.mock.mockito.MockBean
  private BaseMasterDataRequestService baseMasterDataRequestService;

  @org.springframework.boot.test.mock.mockito.MockBean
  private SearchRequestProperties properties;

  @org.springframework.boot.test.mock.mockito.MockBean
  private CurrencyMapper currencyMapper;

  @BeforeEach
  void setUp() {
    when(properties.getSlugValueForCurrency()).thenReturn("currency");
    when(properties.getCurrencyAttributeId()).thenReturn("currencyCode");
    // очистим кеш на всякий случай
    CacheIntrospection.nativeMap(cacheManager, CurrencyService2.CURRENCY_ALL).clear();
  }

  @Test
  void getAll_cachesResult_andStoresUnderALLKey() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(
        CurrencyDto.builder().currencyCode("USD").build(),
        CurrencyDto.builder().currencyCode("EUR").build()
    );

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      var first = currencyCacheOps.getAll();
      var second = currencyCacheOps.getAll();

      // бэкенд вызвался ровно один раз
      verify(baseMasterDataRequestService, times(1))
          .requestDataByAttributes("currency", "currencyCode", null);

      // результат одинаковый и соответствует маппингу
      assertThat(first).isEqualTo(second).containsExactlyElementsOf(mapped);

      // в кеше есть единственный ключ "ALL"
      var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
      assertThat(keys).containsExactly("ALL");

      // «сырой» объект под ключом — тот же список
      @SuppressWarnings("unchecked")
      var raw = (List<CurrencyDto>) CacheIntrospection.rawValue(
          cacheManager, CurrencyService2.CURRENCY_ALL, "ALL");
      assertThat(raw).containsExactlyElementsOf(mapped);
    }
  }

  @Test
  void getAll_thenSecondCall_isHit_byCaffeineStats() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(CurrencyDto.builder().currencyCode("JPY").build());

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      var cache = (CaffeineCache) cacheManager.getCache(CurrencyService2.CURRENCY_ALL);
      var before = cache.getNativeCache().stats();

      currencyCacheOps.getAll(); // miss
      currencyCacheOps.getAll(); // hit

      var after = cache.getNativeCache().stats();

      // проверяем дельту: 1 промах и 1 попадание
      assertThat(after.missCount() - before.missCount()).isEqualTo(1);
      assertThat(after.hitCount() - before.hitCount()).isEqualTo(1);

      verify(baseMasterDataRequestService, times(1))
          .requestDataByAttributes("currency", "currencyCode", null);
    }
  }

  @Test
  void afterManualClear_nextCall_isMiss() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(CurrencyDto.builder().currencyCode("GBP").build());

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      currencyCacheOps.getAll(); // первая загрузка -> кешируем

      // удаляем ровно элемент "ALL" из кеша
      CacheIntrospection.nativeMap(cacheManager, CurrencyService2.CURRENCY_ALL).remove("ALL");

      currencyCacheOps.getAll(); // вторая загрузка -> снова идём в бэкенд

      verify(baseMasterDataRequestService, times(2))
          .requestDataByAttributes("currency", "currencyCode", null);
    }
  }
}


@Test
void loadByCodes_delegatesToBackend_andMaps() {
  // given
  var codes = List.of("USD", "EUR");
  var resp  = new GetItemsSearchResponse();
  var mapped = List.of(
      CurrencyDto.builder().currencyCode("USD").build(),
      CurrencyDto.builder().currencyCode("EUR").build()
  );

  when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", codes))
      .thenReturn(resp);

  try (MockedStatic<BaseMasterDataRequestService> statics =
           mockStatic(BaseMasterDataRequestService.class)) {
    statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
        .thenReturn(mapped);

    // when
    var result = currencyCacheOps.loadByCodes(codes);

    // then
    verify(baseMasterDataRequestService, times(1))
        .requestDataByAttributes("currency", "currencyCode", codes);
    assertThat(result).containsExactlyElementsOf(mapped);

    // метод не должен что-то класть в кеш "ALL"
    var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
    assertThat(keys).isEmpty();
  }
}

@Test
void loadByCodes_calledTwice_callsBackendTwice_andDoesNotCache() {
  // given
  var codes = List.of("JPY");
  var resp  = new GetItemsSearchResponse();
  var mapped = List.of(CurrencyDto.builder().currencyCode("JPY").build());

  when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", codes))
      .thenReturn(resp);

  try (MockedStatic<BaseMasterDataRequestService> statics =
           mockStatic(BaseMasterDataRequestService.class)) {
    statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
        .thenReturn(mapped);

    // when
    var r1 = currencyCacheOps.loadByCodes(codes);
    var r2 = currencyCacheOps.loadByCodes(codes);

    // then: без @Cacheable результат одинаковый, но бэкенд дернулся 2 раза
    assertThat(r1).containsExactlyElementsOf(mapped);
    assertThat(r2).containsExactlyElementsOf(mapped);

    verify(baseMasterDataRequestService, times(2))
        .requestDataByAttributes("currency", "currencyCode", codes);

    // кеш "ALL" не трогаем
    var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
    assertThat(keys).isEmpty();
  }
}



```
