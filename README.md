```java
@ExtendWith(MockitoExtension.class)
class TrackerKafkaProducerTest {

    @Mock
    private KafkaTemplate<String, HistoryNewDto> historyNewTemplate;

    @Mock
    private KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    private TrackerKafkaProducer trackerKafkaProducer;

    @BeforeEach
    void setUp() {
        trackerKafkaProducer = new TrackerKafkaProducer(
                historyNewTemplate,
                plannedDateTemplate
        );

        setField(trackerKafkaProducer, "historyTopic", "tracker_history_test");
        setField(trackerKafkaProducer, "additionalTopic", "tracker_additional_test");
    }

    @Test
    void sendHistory_shouldCallKafkaTemplateSend() {
        UUID entityUuid = UUID.randomUUID();

        HistoryNewDto dto = mock(HistoryNewDto.class);
        when(dto.getEntityUuid()).thenReturn(entityUuid);
        when(dto.getStatus()).thenReturn("DRAFT");

        trackerKafkaProducer.sendHistory(dto);

        verify(historyNewTemplate).send("tracker_history_test", entityUuid.toString(), dto);
        verifyNoInteractions(plannedDateTemplate);
    }

    @Test
    void sendAdditional_shouldCallKafkaTemplateSend() {
        UUID entityUuid = UUID.randomUUID();

        PlannedDateDto dto = mock(PlannedDateDto.class);
        when(dto.getEntityUuid()).thenReturn(entityUuid);

        trackerKafkaProducer.sendAdditional(dto);

        verify(plannedDateTemplate).send("tracker_additional_test", entityUuid.toString(), dto);
        verifyNoInteractions(historyNewTemplate);
    }
}
```
