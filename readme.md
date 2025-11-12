```java

@Test
void handleInvalidLoadedItemKey() {
    // given
    when(messageSource.getMessage(eq("error.invalidCacheKey"), any(), any()))
        .thenReturn("Ошибка при кэшировании элемента");

    WebRequest request = mock(WebRequest.class);
    var ex = new MdaInvalidCacheKeyException("tb_by_code", new Object());

    // when
    var response = globalExceptionHandler.handleInvalidLoadedItemKey(ex, request);

    // then
    assertThat(response).isNotNull();
    assertNotNull(response.getBody());

    MessageObj body = (MessageObj) response.getBody();

    assertEquals("Ошибка при кэшировании элемента",
        body.getMessages().get(0).getMessage());
    assertEquals(AppMessageSemantic.E,
        body.getMessages().get(0).getSemantic());
    assertEquals(HttpStatus.INTERNAL_SERVER_ERROR,
        response.getStatusCode());
}
```
