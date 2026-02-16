```java
@Test
void givenProcessorThrows_whenHandleMessage_thenPropagatesException() {
    // GIVEN
    ConsumerRecord<String, String> consumerRecord =
            new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, "k", "<PutEODPriceNf/>");

    RuntimeException boom = new RuntimeException("boom");
    doThrow(boom).when(recordProcessor).process(eq(consumerRecord), any(RoutesRegistry.class));

    // WHEN
    Throwable thrown = catchThrowable(() -> listener.handleMessage(consumerRecord));

    // THEN
    assertThat(thrown).isSameAs(boom);
    verify(recordProcessor).process(eq(consumerRecord), any(RoutesRegistry.class));
    verifyNoMoreInteractions(recordProcessor);
}
```
