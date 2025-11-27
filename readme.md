```java

@Test
@DisplayName("Get /api/v1/info/country без параметров")
void searchCountriesWithoutCodesReturnsBadRequest() throws Exception {
    MvcResult result = performGet(mockMvc, "/api/v1/info/country").andReturn();

    assertEquals(400, result.getResponse().getStatus());
    assertEquals(
            MissingServletRequestParameterException.class,
            result.getResolvedException().getClass()
    );
}
```
