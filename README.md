```java
@ExtendWith(MockitoExtension.class)
class StsTrackerHistoryServiceTest {

    @Mock
    private TrackerKafkaProducer trackerKafkaProducer;

    @InjectMocks
    private StsTrackerHistoryService stsTrackerHistoryService;

    @Captor
    private ArgumentCaptor<HistoryNewDto> historyCaptor;

    @Test
    void sendCreatedStatus_shouldBuildDtoAndSendHistory() {
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        StsCreatedTrackerHistoryEvent event = org.mockito.Mockito.mock(StsCreatedTrackerHistoryEvent.class);

        // Подставь здесь свой реальный enum статуса и существующее значение
        StsStatus status = StsStatus.DRAFT;

        when(event.entityUuid()).thenReturn(entityUuid);
        when(event.statusId()).thenReturn(status);
        when(event.createdBy()).thenReturn(createdBy);

        stsTrackerHistoryService.sendCreatedStatus(event);

        verify(trackerKafkaProducer).sendHistory(historyCaptor.capture());
        verifyNoMoreInteractions(trackerKafkaProducer);

        HistoryNewDto dto = historyCaptor.getValue();

        assertAll(
                () -> assertEquals(StsTrackerSchemeCodes.STATUS_STS, dto.getCode()),
                () -> assertEquals(entityUuid, dto.getEntityUuid()),
                () -> assertEquals(StsAction.CREATE_STS.getId(), dto.getOperation()),
                () -> assertEquals(status.name(), dto.getStatus()),
                () -> assertNull(dto.getComment()),
                () -> assertEquals(createdBy, dto.getCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getExtCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getUserName()),
                () -> assertNull(dto.getUserPosition())
        );
    }
}
```
