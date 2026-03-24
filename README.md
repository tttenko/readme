```java
@Test
void givenHistoryIntegrationException_whenHandleHistoryIntegrationException_thenReturnBadGatewayResponse() {
    // given
    String exceptionMessage = "Не удалось записать событие в историю операций";
    HistoryIntegrationException exception =
            new HistoryIntegrationException(exceptionMessage, new RuntimeException("fail"));

    // when
    ResponseEntity<Object> response =
            globalExceptionHandler.handleHistoryIntegrationException(exception, webRequest);

    // then
    assertThat(response).isNotNull();
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_GATEWAY);
    assertThat(response.getBody()).isInstanceOf(String.class);
    assertThat(response.getBody()).isEqualTo(exceptionMessage);

    verifyNoInteractions(messageSource);
}



```
