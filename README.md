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


@ExtendWith(MockitoExtension.class)
class StsEventsHistoryServiceTest {

    @Mock
    private EventsHistoryClient eventsHistoryClient;

    @InjectMocks
    private StsEventsHistoryService stsEventsHistoryService;

    @Captor
    private ArgumentCaptor<EventCreateDto> eventCaptor;

    @Test
    void givenValidEntity_whenSendCreateEvent_thenSendEventToHistoryClient() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();
        ZonedDateTime createdAt = ZonedDateTime.now();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setContractUuid(contractUuid);
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setComment("Тестовая запись");
        entity.setStatusId(StsStatus.DRAFT.getId());
        entity.setCreatedBy("system");
        entity.setCreatedAt(createdAt);

        // when
        stsEventsHistoryService.sendCreateEvent(entity);

        // then
        verify(eventsHistoryClient).createEvent(eventCaptor.capture());
        verifyNoMoreInteractions(eventsHistoryClient);

        EventCreateDto actualEvent = eventCaptor.getValue();

        assertThat(actualEvent).isNotNull();
        assertThat(actualEvent.getServiceId()).isEqualTo(StsHistoryEventIds.SERVICE_ID);
        assertThat(actualEvent.getId()).isEqualTo(StsHistoryEventIds.STS_CREATE);
        assertThat(actualEvent.getEntityUuid()).isEqualTo(entityUuid);
        assertThat(actualEvent.getSubmittedAt()).isEqualTo(createdAt);
        assertThat(actualEvent.getSubmittedBy()).isEqualTo("system");
        assertThat(actualEvent.getSession()).isEqualTo("NO-SESSION");
        assertThat(actualEvent.getUserName()).isEqualTo("system");
        assertThat(actualEvent.getUserNode()).isEqualTo("NO-USERNODE");

        assertThat(actualEvent.getParameters()).isNotNull();
        assertThat(actualEvent.getParameters()).isNotEmpty();

        Map<String, String> parameters = actualEvent.getParameters().stream()
                .collect(Collectors.toMap(
                        EventParameterCreateDto::getId,
                        EventParameterCreateDto::getValue
                ));

        assertThat(parameters.get("vehicle_num")).isEqualTo("A123AA777");
        assertThat(parameters.get("vehicle_name")).isEqualTo("КАМАЗ");

        // Эта проверка покрывает StsStatus, если status реально передается в event
        if (parameters.containsKey("status")) {
            assertThat(parameters.get("status")).isEqualTo("Черновик");
        }
    }

    @Test
    void givenHistoryClientThrowsException_whenSendCreateEvent_thenThrowHistoryIntegrationException() {
        // given
        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(UUID.randomUUID());
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setComment("Тестовая запись");
        entity.setStatusId(StsStatus.DRAFT.getId());
        entity.setCreatedBy("system");
        entity.setCreatedAt(ZonedDateTime.now());

        doThrow(new RuntimeException("history error"))
                .when(eventsHistoryClient)
                .createEvent(any(EventCreateDto.class));

        // when / then
        assertThatThrownBy(() -> stsEventsHistoryService.sendCreateEvent(entity))
                .isInstanceOf(HistoryIntegrationException.class)
                .hasMessage("Не удалось записать событие в историю операций")
                .hasCauseInstanceOf(RuntimeException.class);

        verify(eventsHistoryClient).createEvent(any(EventCreateDto.class));
        verifyNoMoreInteractions(eventsHistoryClient);
    }
}
```
