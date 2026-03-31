```java
@Test
void sendHistory_shouldThrowIllegalStateException_whenInterrupted() throws Exception {
    UUID entityUuid = UUID.randomUUID();

    HistoryNewDto dto = mock(HistoryNewDto.class);
    when(dto.getEntityUuid()).thenReturn(entityUuid);

    CompletableFuture<SendResult<String, HistoryNewDto>> future = mock(CompletableFuture.class);

    when(historyNewTemplate.send(anyString(), anyString(), any()))
            .thenReturn(future);

    when(future.get()).thenThrow(new InterruptedException());

    assertThatThrownBy(() -> trackerKafkaProducer.sendHistory(dto))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Failed to send tracker history event");

    verify(historyNewTemplate).send(anyString(), anyString(), eq(dto));
}
2. sendHistory — ExecutionException
@Test
void sendHistory_shouldThrowIllegalStateException_whenExecutionException() throws Exception {
    UUID entityUuid = UUID.randomUUID();

    HistoryNewDto dto = mock(HistoryNewDto.class);
    when(dto.getEntityUuid()).thenReturn(entityUuid);

    CompletableFuture<SendResult<String, HistoryNewDto>> future = mock(CompletableFuture.class);

    when(historyNewTemplate.send(anyString(), anyString(), any()))
            .thenReturn(future);

    when(future.get()).thenThrow(new ExecutionException(new RuntimeException()));

    assertThatThrownBy(() -> trackerKafkaProducer.sendHistory(dto))
            .isInstanceOf(IllegalStateException.class);

    verify(historyNewTemplate).send(anyString(), anyString(), eq(dto));
}
3. sendAdditional — InterruptedException
@Test
void sendAdditional_shouldThrowIllegalStateException_whenInterrupted() throws Exception {
    UUID entityUuid = UUID.randomUUID();

    PlannedDateDto dto = mock(PlannedDateDto.class);
    when(dto.getEntityUuid()).thenReturn(entityUuid);

    CompletableFuture<SendResult<String, PlannedDateDto>> future = mock(CompletableFuture.class);

    when(plannedDateTemplate.send(anyString(), anyString(), any()))
            .thenReturn(future);

    when(future.get()).thenThrow(new InterruptedException());

    assertThatThrownBy(() -> trackerKafkaProducer.sendAdditional(dto))
            .isInstanceOf(IllegalStateException.class);

    verify(plannedDateTemplate).send(anyString(), anyString(), eq(dto));
}
4. sendAdditional — ExecutionException
@Test
void sendAdditional_shouldThrowIllegalStateException_whenExecutionException() throws Exception {
    UUID entityUuid = UUID.randomUUID();

    PlannedDateDto dto = mock(PlannedDateDto.class);
    when(dto.getEntityUuid()).thenReturn(entityUuid);

    CompletableFuture<SendResult<String, PlannedDateDto>> future = mock(CompletableFuture.class);

    when(plannedDateTemplate.send(anyString(), anyString(), any()))
            .thenReturn(future);

    when(future.get()).thenThrow(new ExecutionException(new RuntimeException()));

    assertThatThrownBy(() -> trackerKafkaProducer.sendAdditional(dto))
            .isInstanceOf(IllegalStateException.class);

    verify(plannedDateTemplate).send(anyString(), anyString(), eq(dto));
}
```
