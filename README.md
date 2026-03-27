```java
@ExtendWith(MockitoExtension.class)
class TrackerKafkaProducerTest {

    @Mock
    private KafkaTemplate<String, HistoryNewDto> historyNewTemplate;

    @Mock
    private KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    @InjectMocks
    private TrackerKafkaProducer trackerKafkaProducer;

    @BeforeEach
    void setUp() {
        ReflectionTestUtils.setField(trackerKafkaProducer, "historyTopic", "tracker_history");
        ReflectionTestUtils.setField(trackerKafkaProducer, "additionalTopic", "tracker_additional");
    }

    @Test
    void sendHistory_shouldCallKafkaTemplateSend() {
        UUID entityUuid = UUID.randomUUID();

        HistoryNewDto dto = mock(HistoryNewDto.class);
        when(dto.getEntityUuid()).thenReturn(entityUuid);
        when(dto.getStatus()).thenReturn("DRAFT");

        trackerKafkaProducer.sendHistory(dto);

        verify(historyNewTemplate).send("tracker_history", entityUuid.toString(), dto);
        verifyNoMoreInteractions(historyNewTemplate);
        verifyNoInteractions(plannedDateTemplate);
    }

    @Test
    void sendAdditional_shouldCallKafkaTemplateSend() {
        UUID entityUuid = UUID.randomUUID();

        PlannedDateDto dto = mock(PlannedDateDto.class);
        when(dto.getEntityUuid()).thenReturn(entityUuid);

        trackerKafkaProducer.sendAdditional(dto);

        verify(plannedDateTemplate).send("tracker_additional", entityUuid.toString(), dto);
        verifyNoMoreInteractions(plannedDateTemplate);
        verifyNoInteractions(historyNewTemplate);
    }
}


```
