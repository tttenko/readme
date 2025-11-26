```java
@Test
void handleIncorrectRequestException_shouldReturnBadRequestWithLocalizedMessage() {
    // given
    when(messageSource.getMessage(eq("error.incorrectRequest"), any(), any()))
            .thenReturn("Некорректный запрос к Адаптеру МД");

    WebRequest request = mock(WebRequest.class);
    var ex = new MdaIncorrectRequestException();

    // when
    var response = globalExceptionHandler.handleMdaIncorrectRequestException(ex, request);

    // then
    assertThat(response).isNotNull();
    assertNotNull(response.getBody());

    MessageObj body = (MessageObj) response.getBody();

    assertEquals("Некорректный запрос к Адаптеру МД",
            body.getMessages().get(0).getMessage());
    assertEquals(AppMessageSemantic.E,
            body.getMessages().get(0).getSemantic());
    assertEquals(HttpStatus.BAD_REQUEST, response.getStatusCode());
}
```
