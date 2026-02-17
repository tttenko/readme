```java/**
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CurrencyServiceTest {

    @Mock
    private CurrencyCacheOps currencyCacheOps;

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @Mock
    private FxRateRepository fxRateRepository;

    @InjectMocks
    private CurrencyService currencyService2;

    @Test
    void getAllCurrencies_thenLoadAllFromCache() {
        // given
        List<CurrencyDto> all = List.of(
            mock(CurrencyDto.class),
            mock(CurrencyDto.class)
        );

        when(currencyCacheOps.loadAllCurrencies()).thenReturn(all);

        // when
        List<CurrencyDto> result = currencyService2.getAllCurrencies();

        // then
        assertThat(result).isEqualTo(all);
        verify(currencyCacheOps).loadAllCurrencies();
        verifyNoInteractions(cacheGetOrLoadService, fxRateRepository);
    }

    @Test
    void getCurrenciesByCodes_whenEmpty_thenReturnEmptyAndDoNotCallDependencies() {
        // given
        List<String> codes = List.of();

        // when
        List<CurrencyDto> result = currencyService2.getCurrenciesByCodes(codes);

        // then
        assertThat(result).isEmpty();
        verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService, fxRateRepository);
    }

    @Test
    void getCurrenciesByCodes_whenNotEmpty_thenUseCacheGetOrLoadService() {
        // given
        List<String> codes = List.of("USD", "EUR");
        List<CurrencyDto> loaded = List.of(
            mock(CurrencyDto.class),
            mock(CurrencyDto.class)
        );

        when(cacheGetOrLoadService.fetchData(
            CurrencyService.CURRENCY_BY_CODE,
            codes
        )).thenReturn(loaded);

        // when
        List<CurrencyDto> result = currencyService2.getCurrenciesByCodes(codes);

        // then
        assertThat(result).isEqualTo(loaded);
        verify(cacheGetOrLoadService).fetchData(CurrencyService.CURRENCY_BY_CODE, codes);
        verifyNoInteractions(currencyCacheOps, fxRateRepository);
    }

    // ---------------- NEW TESTS for getCurrencyRate ----------------

    @Test
    void getCurrencyRate_whenRateExists_thenReturnSingleDto_andCallRepoWithExclusiveDate() {
        // given
        LocalDate date = LocalDate.of(2026, 2, 17);
        String from = "USD";
        String to = "RUB";
        LocalDateTime expectedExclusive = LocalDateTime.of(2026, 2, 18, 0, 0);

        FxRateEntity entity = mock(FxRateEntity.class);

        ZonedDateTime useDate = ZonedDateTime.of(2026, 2, 17, 0, 0, 0, 0, ZoneOffset.UTC);
        BigDecimal value = new BigDecimal("1");
        BigDecimal lotSize = new BigDecimal("6"); // 1/6 = 0.166666... -> 0.1667 (scale 4, HALF_UP)

        when(entity.getFromCurrencyCode()).thenReturn(from);
        when(entity.getFromCurrencyIsoNum()).thenReturn("840");
        when(entity.getToCurrencyCode()).thenReturn(to);
        when(entity.getToCurrencyIsoNum()).thenReturn("643");
        when(entity.getUseDate()).thenReturn(useDate);
        when(entity.getValue()).thenReturn(value);
        when(entity.getLotSize()).thenReturn(lotSize);

        when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.of(entity));

        // when
        List<CurrencyRateDto> result = currencyService2.getCurrencyRate(date, from, to);

        // then
        assertThat(result).hasSize(1);

        CurrencyRateDto dto = result.get(0);
        assertThat(dto.getCode1()).isEqualTo(from);
        assertThat(dto.getIsoNum1()).isEqualTo("840");
        assertThat(dto.getCode2()).isEqualTo(to);
        assertThat(dto.getIsoNum2()).isEqualTo("643");
        assertThat(dto.getDate()).isEqualTo(date);
        assertThat(dto.getValue()).isEqualByComparingTo("1");
        assertThat(dto.getLotSize()).isEqualByComparingTo("6");
        assertThat(dto.getCurrencyRate()).isEqualByComparingTo("0.1667");

        verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);
        verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
    }

    @Test
    void getCurrencyRate_whenNoRate_thenReturnEmptyList() {
        // given
        LocalDate date = LocalDate.of(2026, 2, 17);
        String from = "USD";
        String to = "RUB";
        LocalDateTime expectedExclusive = LocalDateTime.of(2026, 2, 18, 0, 0);

        when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.empty());

        // when
        List<CurrencyRateDto> result = currencyService2.getCurrencyRate(date, from, to);

        // then
        assertThat(result).isEmpty();
        verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);
        verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
    }

    @Test
    void getCurrencyRate_whenLotSizeIsZero_thenThrowArithmeticException() {
        // given
        LocalDate date = LocalDate.of(2026, 2, 17);
        String from = "USD";
        String to = "RUB";
        LocalDateTime expectedExclusive = LocalDateTime.of(2026, 2, 18, 0, 0);

        FxRateEntity entity = mock(FxRateEntity.class);

        when(entity.getFromCurrencyCode()).thenReturn(from);
        when(entity.getFromCurrencyIsoNum()).thenReturn("840");
        when(entity.getToCurrencyCode()).thenReturn(to);
        when(entity.getToCurrencyIsoNum()).thenReturn("643");
        when(entity.getUseDate()).thenReturn(ZonedDateTime.of(2026, 2, 17, 0, 0, 0, 0, ZoneOffset.UTC));
        when(entity.getValue()).thenReturn(new BigDecimal("10"));
        when(entity.getLotSize()).thenReturn(BigDecimal.ZERO);

        when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.of(entity));

        // when / then
        assertThatThrownBy(() -> currencyService2.getCurrencyRate(date, from, to))
            .isInstanceOf(ArithmeticException.class);

        verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);
        verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
    }
}
```
