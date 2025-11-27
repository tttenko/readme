```java

@Test
@DisplayName("test GET {host}/api/v1/info/country?countryCode=null")
void searchCountryByCode_whenCountryCodeEmpty_thenBadRequest() throws Exception {
    // when
    MvcResult result = MvcTestUtils.performGet(
            mockMvc,
            "/api/v1/info/country",
            "countryCode", ""          // есть параметр, но пустой
    );

    // then
    assertEquals(400, result.getResponse().getStatus());
    assertEquals(
            ConstraintViolationException.class,   // jakarta.validation.ConstraintViolationException
            result.getResolvedException().getClass()
    );
}
```
