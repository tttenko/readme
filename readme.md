```java

@Test
void handleMdaBatchLoadException() {
    // given: локализованное сообщение, которое вернёт MessageSource.
    // (у тебя handleMdaBatchLoadException вызывает getErrorMessageForException(ex),
    // а тот для "не webclient" исключений берёт код "error.retrievingError")
    when(messageSource.getMessage(eq("error.retrievingError"), any(), any()))
            .thenReturn("Ошибка при получении данных");

    WebRequest request = mock(WebRequest.class);
    var ex = new MdaBatchLoadException(new RuntimeException("boom"));

    // when
    var response = globalExceptionHandler.handleMdaBatchLoadException(ex, request);

    // then
    assertThat(response).isNotNull();
    assertNotNull(response.getBody());

    MessageObj body = (MessageObj) response.getBody();
    assertEquals("Ошибка при получении данных", body.getMessages().get(0).getMessage());
    assertEquals(AppMessageSemantic.E, body.getMessages().get(0).getSemantic());

    // статус 503 (Service Unavailable)
    assertEquals(HttpStatus.SERVICE_UNAVAILABLE, response.getStatusCode());
    // либо, если в остальных тестах используешь HttpStatusCode:
    // assertEquals(HttpStatusCode.valueOf(503), response.getStatusCode());
}

```
