```java/**
private static FxRateEntity mockFxRateEntityForRate(
        String from, String iso1,
        String to, String iso2,
        LocalDate useDate,
        BigDecimal value,
        BigDecimal lotSize
) {
    FxRateEntity e = mock(FxRateEntity.class);

    when(e.getFromCurrencyCode()).thenReturn(from);
    when(e.getFromCurrencyIsoNum()).thenReturn(iso1);
    when(e.getToCurrencyCode()).thenReturn(to);
    when(e.getToCurrencyIsoNum()).thenReturn(iso2);

    when(e.getUseDate()).thenReturn(useDate.atStartOfDay(ZoneOffset.UTC));
    when(e.getValue()).thenReturn(value);
    when(e.getLotSize()).thenReturn(lotSize);

    return e;
}

@Test
@DisplayName("тест GET {host}/api/v1/info/currency/rate?date=2026-02-17&fromCurrencyCode=USD&toCurrencyCode=RUB -> 1 элемент")
void getCurrencyRate_whenRateExists_thenReturnOneItem() throws Exception {
    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";
    LocalDateTime expectedExclusive = date.plusDays(1).atStartOfDay();

    FxRateEntity entity = mockFxRateEntityForRate(
            from, "840",
            to, "643",
            date,
            new BigDecimal("1"),
            new BigDecimal("6") // 1/6 = 0.1667
    );

    when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.of(entity));

    // when
    MvcResult mvcResult = mockMvc.perform(get("/api/v1/info/currency/rate")
                    .param("date", "2026-02-17")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then
    verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);

    // без завязки на структуру ResultObj — проверяем, что в ответе реально есть нужные значения
    assertThat(body).contains("USD");
    assertThat(body).contains("RUB");
    assertThat(body).contains("0.1667"); // currencyRate
}

@Test
@DisplayName("тест GET {host}/api/v1/info/currency/rate -> если курса нет, возвращается пустой список")
void getCurrencyRate_whenNoRate_thenReturnEmptyList() throws Exception {
    // given
    LocalDate date = LocalDate.of(2026, 2, 17);
    String from = "USD";
    String to = "RUB";
    LocalDateTime expectedExclusive = date.plusDays(1).atStartOfDay();

    when(fxRateRepository.findLatestRate(from, to, expectedExclusive))
            .thenReturn(Optional.empty());

    // when
    MvcResult mvcResult = mockMvc.perform(get("/api/v1/info/currency/rate")
                    .param("date", "2026-02-17")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then
    verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
    assertThat(body).contains("[]"); // где-то в JSON точно будет пустой список
}

@Test
@DisplayName("тест GET {host}/api/v1/info/currency/rate -> без обязательного параметра date получаем 400")
void getCurrencyRate_whenMissingDateParam_thenBadRequest() throws Exception {
    mockMvc.perform(get("/api/v1/info/currency/rate")
                    .param("fromCurrencyCode", "USD")
                    .param("toCurrencyCode", "RUB"))
            .andExpect(status().isBadRequest());

    verifyNoInteractions(fxRateRepository);
}
```
