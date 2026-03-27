```java
@ExtendWith(MockitoExtension.class)
class StsEventsHistoryServiceTest {

    private static final UUID AUTHOR_UUID =
            UUID.fromString("00000000-0000-0000-0000-000000000001");

    private static final String FULL_NAME = "Сеньшин Осман Людовикович";

    @Mock
    private EventsHistoryClient eventsHistoryClient;

    @InjectMocks
    private StsEventsHistoryService stsEventsHistoryService;

    @Captor
    private ArgumentCaptor<EventCreateDto> eventCaptor;

    @BeforeEach
    void setUpSecurityContext() {
        AuthorizedUser authorizedUser = mock(AuthorizedUser.class);
        when(authorizedUser.getFullName()).thenReturn(FULL_NAME);

        Authentication authentication =
                new UsernamePasswordAuthenticationToken(authorizedUser, null, List.of());

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);
    }

    @AfterEach
    void clearSecurityContext() {
        SecurityContextHolder.clearContext();
    }

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
        entity.setStatusId(StsStatus.DRAFT);
        entity.setCreatedBy(AUTHOR_UUID);
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

        // ВАЖНО:
        // submittedBy сейчас в сервисе берется из entity.getCreatedBy().toString()
        assertThat(actualEvent.getSubmittedBy()).isEqualTo(AUTHOR_UUID.toString());

        // userName берется из AuthorizedUser.getFullName()
        assertThat(actualEvent.getUserName()).isEqualTo(FULL_NAME);

        assertThat(actualEvent.getSession()).isEqualTo("NO-SESSION");
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
        entity.setStatusId(StsStatus.DRAFT);
        entity.setCreatedBy(AUTHOR_UUID);
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
