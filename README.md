```java
@Test
void givenOutboxMessageActionException_whenHandleOutboxMessageActionException_thenReturnInternalServerErrorResponse() {
    // given
    String exceptionMessage = "Unable to serialize payload for outbox message";
    String expectedMessage = "Внутренняя ошибка при подготовке события для отправки";

    OutboxMessageActionException exception =
            new OutboxMessageActionException(exceptionMessage, new RuntimeException("Serialization error"));

    // when
    ResponseEntity<Object> response =
            globalExceptionHandler.handleOutboxMessageActionException(exception, webRequest);

    // then
    assertThat(response).isNotNull();
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.INTERNAL_SERVER_ERROR);
    assertThat(response.getBody()).isInstanceOf(String.class);
    assertThat(response.getBody()).isEqualTo(expectedMessage);

    verifyNoInteractions(messageSource);
}
```
