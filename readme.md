```java/**
@Test
void getCurrencyRate_whenLotSizeIsZero_thenThrowArithmeticException() {
    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";
    LocalDateTime expectedExclusive = date.plusDays(1).atStartOfDay();

    FxRateEntity entity = mock(FxRateEntity.class);

    // Минимально необходимое для падения на divide:
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
