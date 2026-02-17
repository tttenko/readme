```java/**
@Test
@DisplayName("тест GET {host}/api/v1/info/currency/rate?date=2026-02-17&fromCurrencyCode=USD&toCurrencyCode=RUB -> 1 элемент")
void getCurrencyRate_whenRateExists_thenReturnOneItem() throws Exception {
    // given
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
    MvcResult mvcResult = mockMvc.perform(get("/api/v1/info/currency/rate")
                    .param("date", "2026-02-17")
                    .param("fromCurrencyCode", from)
                    .param("toCurrencyCode", to))
            .andExpect(status().isOk())
            .andReturn();

    // then
    verify(fxRateRepository).findLatestRate(from, to, expectedExclusive);

    String body = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);

    // Проверяем ключевые значения в ответе (не завязываемся на структуру ResultObj)
    assertThat(body).contains("USD");
    assertThat(body).contains("RUB");
    assertThat(body).contains("0.1667"); // currencyRate = 1/6 с scale=4, HALF_UP
}
```
