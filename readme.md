```java/**
@Test
@DisplayName("тест GET /api/v1/info/currency/(currency)/rate -> 1 элемент")
void getCurrencyRate_whenRateExists_thenReturnOneItem() throws Exception {
    // given
    // ВАЖНО: путь берём такой, какой получается из ваших аннотаций:
    // /api/v1/info/currency + /currency/rate = /api/v1/info/currency/currency/rate
    String url = "/api/v1/info/currency/currency/rate";

    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";
    LocalDateTime expectedExclusive = date.plusDays(1).atStartOfDay();

    ZonedDateTime useDate = date.atStartOfDay(ZoneOffset.UTC);

    FxRateEntity entity = mockFxRateEntityForRate(
            from, "840",
            to, "643",
            useDate,
            new BigDecimal("1"),
            new BigDecimal("6") // 1/6 = 0.1667
    );

    when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.of(entity));

    // when
    MvcResult mvcResult = mockMvc.perform(get(url)
                    .param("date", "17.02.2026")          // dd.MM.yyyy
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then
    verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
    assertThat(body).contains("USD");
    assertThat(body).contains("RUB");
    assertThat(body).contains("0.1667");
}

@Test
@DisplayName("тест GET /api/v1/info/currency/(currency)/rate -> если курса нет, возвращается пустой список")
void getCurrencyRate_whenNoRate_thenReturnEmptyList() throws Exception {
    String url = "/api/v1/info/currency/currency/rate";

    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";
    LocalDateTime expectedExclusive = date.plusDays(1).atStartOfDay();

    when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.empty());

    MvcResult mvcResult = mockMvc.perform(get(url)
                    .param("date", "17.02.2026")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
    assertThat(body).contains("[]");
}


```
