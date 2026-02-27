```java/**
@Test
@DisplayName("тест GET /api/v1/info/currency/(currency)/rate -> 1 элемент")
void getCurrencyRate_whenRateExists_thenReturnOneItem() throws Exception {

    // given
    String url = "/api/v1/info/currency/rate";

    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";

    ZonedDateTime expectedExclusive = date.plusDays(1)
            .atStartOfDay(ZoneId.systemDefault());
    ZonedDateTime useDate = date.atStartOfDay(ZoneId.systemDefault());

    FxRateEntity entity = mockFxRateEntityForRate(
            from, "840",
            to, "643",
            useDate,
            new BigDecimal("1"),
            new BigDecimal("6")
    );

    when(fxRateRepository.findLatestRate(eq(from), eq(to), eq(expectedExclusive), any(Pageable.class)))
            .thenReturn(java.util.Optional.of(entity));

    // when
    MvcResult mvcResult = mockMvc.perform(get(url)
                    .param("date", "2026-02-17")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then: проверяем, что pageable передали (page=0, size=1)
    ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);

    verify(fxRateRepository).findLatestRate(eq(from), eq(to), eq(expectedExclusive), pageableCaptor.capture());

    Pageable passedPageable = pageableCaptor.getValue();
    assertThat(passedPageable.getPageNumber()).isEqualTo(0);
    assertThat(passedPageable.getPageSize()).isEqualTo(1);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
    assertThat(body).contains("USD", "RUB", "0.1667");
}

@Test
@DisplayName("тест GET /api/v1/info/currency/(currency)/rate -> если курса нет, возвращается пустой список")
void getCurrencyRate_whenNoRate_thenReturnEmptyList() throws Exception {

    // given
    String url = "/api/v1/info/currency/rate";

    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";

    ZonedDateTime expectedExclusive = date.plusDays(1)
            .atStartOfDay(ZoneId.systemDefault());

    when(fxRateRepository.findLatestRate(eq(from), eq(to), eq(expectedExclusive), any(Pageable.class)))
            .thenReturn(java.util.Optional.empty());

    // when
    MvcResult mvcResult = mockMvc.perform(get(url)
                    .param("date", "2026-02-17")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then
    ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);

    verify(fxRateRepository).findLatestRate(eq(from), eq(to), eq(expectedExclusive), pageableCaptor.capture());

    Pageable passedPageable = pageableCaptor.getValue();
    assertThat(passedPageable.getPageNumber()).isEqualTo(0);
    assertThat(passedPageable.getPageSize()).isEqualTo(1);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
    assertThat(body).contains("[]");
}
```
