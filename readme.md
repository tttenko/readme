```java/**
@Test
void getCurrencyRate_whenRateExists_thenReturnSingleDto_andCallRepoWithExclusiveDate() {

    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";

    ZonedDateTime expectedExclusive = LocalDateTime
            .of(2026, 2, 18, 0, 0)
            .atZone(ZoneId.systemDefault());

    FxRateEntity entity = mock(FxRateEntity.class);

    ZonedDateTime useDate = LocalDateTime
            .of(2026, 2, 17, 0, 0, 0, 0)
            .atZone(ZoneId.systemDefault());

    BigDecimal value = new BigDecimal("1");
    BigDecimal lotSize = new BigDecimal("6");

    when(entity.getFromCurrencyCode()).thenReturn(from);
    when(entity.getFromCurrencyIsoNum()).thenReturn("840");
    when(entity.getToCurrencyCode()).thenReturn(to);
    when(entity.getToCurrencyIsoNum()).thenReturn("643");
    when(entity.getUseDate()).thenReturn(useDate);
    when(entity.getValue()).thenReturn(value);
    when(entity.getLotSize()).thenReturn(lotSize);

    when(fxRateRepository.findLatestRate(eq(from), eq(to), eq(expectedExclusive), any(Pageable.class)))
            .thenReturn(Optional.of(entity));

    // when
    List<CurrencyRateDto> result = currencyService.getCurrencyRate(date, from, to);

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

    ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);
    verify(fxRateRepository).findLatestRate(eq(from), eq(to), eq(expectedExclusive), pageableCaptor.capture());

    Pageable passed = pageableCaptor.getValue();
    assertThat(passed.getPageNumber()).isEqualTo(0);
    assertThat(passed.getPageSize()).isEqualTo(1);

    verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
}

@Test
void getCurrencyRate_whenNoRate_thenReturnEmptyList() {

    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";

    ZonedDateTime expectedExclusive = LocalDateTime
            .of(2026, 2, 18, 0, 0)
            .atZone(ZoneId.systemDefault());

    when(fxRateRepository.findLatestRate(eq(from), eq(to), eq(expectedExclusive), any(Pageable.class)))
            .thenReturn(Optional.empty());

    // when
    List<CurrencyRateDto> result = currencyService.getCurrencyRate(date, from, to);

    // then
    assertThat(result).isEmpty();

    ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);
    verify(fxRateRepository).findLatestRate(eq(from), eq(to), eq(expectedExclusive), pageableCaptor.capture());

    Pageable passed = pageableCaptor.getValue();
    assertThat(passed.getPageNumber()).isEqualTo(0);
    assertThat(passed.getPageSize()).isEqualTo(1);

    verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
}

@Test
void getCurrencyRate_whenLotSizeIsZero_thenThrowArithmeticException() {

    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";

    ZonedDateTime expectedExclusive = date.plusDays(1)
            .atStartOfDay(ZoneId.systemDefault());

    FxRateEntity entity = mock(FxRateEntity.class);

    when(entity.getValue()).thenReturn(new BigDecimal("10"));
    when(entity.getLotSize()).thenReturn(BigDecimal.ZERO);

    when(fxRateRepository.findLatestRate(eq(from), eq(to), eq(expectedExclusive), any(Pageable.class)))
            .thenReturn(Optional.of(entity));

    // when / then
    assertThatThrownBy(() -> currencyService.getCurrencyRate(date, from, to))
            .isInstanceOf(ArithmeticException.class);

    ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);
    verify(fxRateRepository).findLatestRate(eq(from), eq(to), eq(expectedExclusive), pageableCaptor.capture());

    Pageable passed = pageableCaptor.getValue();
    assertThat(passed.getPageNumber()).isEqualTo(0);
    assertThat(passed.getPageSize()).isEqualTo(1);

    verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
}
```
