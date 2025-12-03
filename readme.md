```java
@Test
void handleBadRequestException() {
    // given
    String validationMsg = "countryCode не должен быть пустым";
    String expectedMessage = "Некорректный запрос к Адаптеру МД: " + validationMsg;

    // messageSource вернёт уже финальную строку
    when(messageSource.getMessage(eq("error.incorrectRequest"), any(), any()))
            .thenReturn(expectedMessage);

    @SuppressWarnings("unchecked")
    ConstraintViolation<?> violation = mock(ConstraintViolation.class);
    when(violation.getMessage()).thenReturn(validationMsg);

    ConstraintViolationException ex = mock(ConstraintViolationException.class);
    when(ex.getConstraintViolations()).thenReturn(Set.of(violation));

    WebRequest request = mock(WebRequest.class);

    // when
    ResponseEntity<Object> response =
            globalExceptionHandler.handleBadRequestException(ex, request);

    // then
    assertThat(response).isNotNull();
    assertNotNull(response.getBody());

    MessageObj body = (MessageObj) response.getBody();

    assertEquals(expectedMessage, body.getMessages().get(0).getMessage());
    assertEquals(AppMessageSemantic.E, body.getMessages().get(0).getSemantic());
    assertEquals(HttpStatusCode.valueOf(400), response.getStatusCode());
}

```
