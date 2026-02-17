```java/**
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class FxRateIngestServiceTest {

    @Mock
    private FxRateRepository repo;

    @Mock
    private FxRateEntityMapper fxRateEntityMapper;

    @InjectMocks
    private FxRateIngestService service;

    @Test
    void ingest_whenMessageIsNull_shouldThrowIllegalArgumentException() {
        assertThatThrownBy(() -> service.ingest(null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("PutEodPriceNFDto равно null");

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRequestUidIsNull_shouldThrowInvalidCurrencyRateXmlException() {
        PutEodPriceNFDto message = msg(null, LocalDateTime.now(), List.of(mock(FxRateXmlDto.class)));

        assertThatThrownBy(() -> service.ingest(message))
            .isInstanceOf(InvalidCurrencyRateXmlException.class)
            .hasMessageContaining("отсутствует обязательный идентификатор запроса RqUID");

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRequestUidIsBlank_shouldThrowInvalidCurrencyRateXmlException() {
        PutEodPriceNFDto message = msg("   \t\n", LocalDateTime.now(), List.of(mock(FxRateXmlDto.class)));

        assertThatThrownBy(() -> service.ingest(message))
            .isInstanceOf(InvalidCurrencyRateXmlException.class)
            .hasMessageContaining("отсутствует обязательный идентификатор запроса RqUID");

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRequestTimeIsNull_shouldThrowInvalidCurrencyRateXmlException_andIncludeRqUidInMessage() {
        PutEodPriceNFDto message = msg("REQ-123", null, List.of(mock(FxRateXmlDto.class)));

        assertThatThrownBy(() -> service.ingest(message))
            .isInstanceOf(InvalidCurrencyRateXmlException.class)
            .hasMessageContaining("отсутствует обязательное время запроса RqTm")
            .hasMessageContaining("RqUID=REQ-123");

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRatesEmpty_shouldDoNothing() {
        PutEodPriceNFDto message = msg("REQ-1", LocalDateTime.now(), List.of());

        service.ingest(message);

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRatesPresent_shouldMapEachRate_andCallUpsertForEachRate() {
        Fixture f = fixtureTwoRates();

        service.ingest(f.message);

        verify(fxRateEntityMapper).toEntity(f.requestUid, f.requestTime, f.rate1);
        verify(fxRateEntityMapper).toEntity(f.requestUid, f.requestTime, f.rate2);

        verify(repo, times(2)).upsert(any(ExchangeRateUpsertParams.class));

        verifyNoMoreInteractions(repo, fxRateEntityMapper);
    }

    @Test
    void ingest_whenRatesPresent_shouldPassExpectedParamsToUpsert() {
        Fixture f = fixtureTwoRates();

        service.ingest(f.message);

        ArgumentCaptor<ExchangeRateUpsertParams> captor =
            ArgumentCaptor.forClass(ExchangeRateUpsertParams.class);

        verify(repo, times(2)).upsert(captor.capture());

        List<ExchangeRateUpsertParams> actual = captor.getAllValues();
        assertThat(actual).hasSize(2);

        assertThat(actual.get(0)).usingRecursiveComparison().isEqualTo(f.expected1);
        assertThat(actual.get(1)).usingRecursiveComparison().isEqualTo(f.expected2);
    }

    /**
     * (Опционально) фиксирует текущее поведение: если message.fxRates == null, будет NPE на for-each.
     * Если вы позже добавите валидацию rates == null, этот тест нужно будет поменять.
     */
    @Test
    void ingest_whenRatesIsNull_shouldThrowNullPointerException_currentBehavior() {
        PutEodPriceNFDto message = msg("REQ-1", LocalDateTime.now(), null);

        assertThatThrownBy(() -> service.ingest(message))
            .isInstanceOf(NullPointerException.class);

        verifyNoInteractions(repo, fxRateEntityMapper);
    }

    // ---------------- helpers ----------------

    private Fixture fixtureTwoRates() {
        String requestUid = "REQ-777";
        LocalDateTime requestTime = LocalDateTime.of(2026, 2, 17, 10, 11, 12);

        FxRateXmlDto rate1 = mock(FxRateXmlDto.class);
        FxRateXmlDto rate2 = mock(FxRateXmlDto.class);

        FxRateEntity entity1 = mockEntity(
            requestUid, requestTime,
            "SPOT",
            "USD", 840,
            "RUB", 643,
            LocalDate.of(2026, 2, 17),
            1,
            new BigDecimal("91.1234")
        );

        FxRateEntity entity2 = mockEntity(
            requestUid, requestTime,
            "SPOT",
            "EUR", 978,
            "RUB", 643,
            LocalDate.of(2026, 2, 17),
            1,
            new BigDecimal("99.9900")
        );

        when(fxRateEntityMapper.toEntity(requestUid, requestTime, rate1)).thenReturn(entity1);
        when(fxRateEntityMapper.toEntity(requestUid, requestTime, rate2)).thenReturn(entity2);

        PutEodPriceNFDto message = msg(requestUid, requestTime, List.of(rate1, rate2));

        ExchangeRateUpsertParams expected1 = new ExchangeRateUpsertParams(
            requestUid, requestTime, "SPOT",
            "USD", 840,
            "RUB", 643,
            LocalDate.of(2026, 2, 17),
            1,
            new BigDecimal("91.1234")
        );

        ExchangeRateUpsertParams expected2 = new ExchangeRateUpsertParams(
            requestUid, requestTime, "SPOT",
            "EUR", 978,
            "RUB", 643,
            LocalDate.of(2026, 2, 17),
            1,
            new BigDecimal("99.9900")
        );

        return new Fixture(requestUid, requestTime, rate1, rate2, message, expected1, expected2);
    }

    private static class Fixture {
        final String requestUid;
        final LocalDateTime requestTime;
        final FxRateXmlDto rate1;
        final FxRateXmlDto rate2;
        final PutEodPriceNFDto message;
        final ExchangeRateUpsertParams expected1;
        final ExchangeRateUpsertParams expected2;

        private Fixture(String requestUid,
                        LocalDateTime requestTime,
                        FxRateXmlDto rate1,
                        FxRateXmlDto rate2,
                        PutEodPriceNFDto message,
                        ExchangeRateUpsertParams expected1,
                        ExchangeRateUpsertParams expected2) {
            this.requestUid = requestUid;
            this.requestTime = requestTime;
            this.rate1 = rate1;
            this.rate2 = rate2;
            this.message = message;
            this.expected1 = expected1;
            this.expected2 = expected2;
        }
    }

    private static PutEodPriceNFDto msg(String uid, LocalDateTime time, List<FxRateXmlDto> rates) {
        PutEodPriceNFDto dto = new PutEodPriceNFDto();
        dto.rquid = uid;
        dto.rqtm = time;
        dto.fxRates = rates;
        return dto;
    }

    private static FxRateEntity mockEntity(
        String requestUid,
        LocalDateTime requestTime,
        String fxRateSubType,
        String fromCode,
        Integer fromIsoNum,
        String toCode,
        Integer toIsoNum,
        LocalDate useDate,
        Integer lotSize,
        BigDecimal value
    ) {
        FxRateEntity entity = mock(FxRateEntity.class);

        when(entity.getRequestUid()).thenReturn(requestUid);
        when(entity.getRequestTime()).thenReturn(requestTime);
        when(entity.getFxRateSubType()).thenReturn(fxRateSubType);

        when(entity.getFromCurrencyCode()).thenReturn(fromCode);
        when(entity.getFromCurrencyIsoNum()).thenReturn(fromIsoNum);

        when(entity.getToCurrencyCode()).thenReturn(toCode);
        when(entity.getToCurrencyIsoNum()).thenReturn(toIsoNum);

        when(entity.getUseDate()).thenReturn(useDate);
        when(entity.getLotSize()).thenReturn(lotSize);
        when(entity.getValue()).thenReturn(value);

        return entity;
    }
}

```
