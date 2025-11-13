```java

@Test
void handleCacheNotFound() {
    // given
    when(messageSource.getMessage(eq("error.cacheNotFound"), any(), any()))
        .thenReturn("Кэш недоступен или не сконфигурирован");

    WebRequest request = mock(WebRequest.class);
    var ex = new MdaCacheNotFoundException("tb_by_code");

    // when
    var response = globalExceptionHandler.handleCacheNotFound(ex, request);

    // then
    assertThat(response).isNotNull();
    assertNotNull(response.getBody());

    MessageObj body = (MessageObj) response.getBody();
    assertEquals("Кэш недоступен или не сконфигурирован",
        body.getMessages().get(0).getMessage());
    assertEquals(AppMessageSemantic.E,
        body.getMessages().get(0).getSemantic());
    assertEquals(HttpStatus.INTERNAL_SERVER_ERROR,   // или SERVICE_UNAVAILABLE, если так решили
        response.getStatusCode());
}


```
