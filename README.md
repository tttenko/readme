```java

@Test
void givenConstraintViolationException_whenHandleBadRequestException_thenReturnBadRequestResponse() {
    // given
    String validationMessage = "Параметр contractUuid не должен быть null";
    String localizedMessage = "Некорректный запрос: " + validationMessage;

    when(constraintViolation.getMessage()).thenReturn(validationMessage);
    when(messageSource.getMessage(
            eq("error.incorrectRequest"),
            eq(new Object[]{validationMessage}),
            any(Locale.class)
    )).thenReturn(localizedMessage);

    ConstraintViolationException exception =
            new ConstraintViolationException(Set.of(constraintViolation));

    // when
    ResponseEntity<Object> response =
            globalExceptionHandler.handleBadRequestException(exception, webRequest);

    // then
    assertThat(response).isNotNull();
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    assertThat(response.getBody()).isInstanceOf(MessageObj.class);

    MessageObj body = (MessageObj) response.getBody();
    assertThat(body.getMessages()).hasSize(1);
    assertThat(body.getMessages().get(0).getMessage()).isEqualTo(localizedMessage);

    verify(messageSource).getMessage(
            eq("error.incorrectRequest"),
            eq(new Object[]{validationMessage}),
            any(Locale.class)
    );
    verifyNoMoreInteractions(messageSource);
}
```
