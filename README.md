```java
@ExtendWith(MockitoExtension.class)
class TrackerKafkaProducerTest {

    @Mock
    private KafkaTemplate<String, HistoryNewDto> historyNewTemplate;

    @Mock
    private KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    @InjectMocks
    private TrackerKafkaProducer trackerKafkaProducer;

    @Captor
    private ArgumentCaptor<ProducerRecord<String, HistoryNewDto>> historyRecordCaptor;

    @Captor
    private ArgumentCaptor<ProducerRecord<String, PlannedDateDto>> plannedDateRecordCaptor;

    private static final String HISTORY_TOPIC = "tracker_history";
    private static final String ADDITIONAL_TOPIC = "tracker_additional";

    @BeforeEach
    void setUp() {
        // Устанавливаем значения топиков через reflection, так как они из @Value
        ReflectionTestUtils.setField(trackerKafkaProducer, "historyTopic", HISTORY_TOPIC);
        ReflectionTestUtils.setField(trackerKafkaProducer, "additionalTopic", ADDITIONAL_TOPIC);
    }

    @Test
    void sendHistory_ShouldSendMessageToKafkaWithCorrectKeyAndValue() {
        // given
        UUID entityUuid = UUID.randomUUID();
        HistoryNewDto dto = HistoryNewDto.builder()
                .entityUuid(entityUuid)
                .status("ACTIVE")
                .operation("CREATE")
                .build();

        // when
        trackerKafkaProducer.sendHistory(dto);

        // then
        verify(historyNewTemplate).send(historyRecordCaptor.capture());
        
        ProducerRecord<String, HistoryNewDto> record = historyRecordCaptor.getValue();
        
        assertThat(record.topic()).isEqualTo(HISTORY_TOPIC);
        assertThat(record.key()).isEqualTo(entityUuid.toString());
        assertThat(record.value()).isSameAs(dto);
    }

    @Test
    void sendHistory_WhenDtoHasEntityUuid_ShouldUseUuidAsKey() {
        // given
        UUID expectedUuid = UUID.randomUUID();
        HistoryNewDto dto = HistoryNewDto.builder()
                .entityUuid(expectedUuid)
                .status("ACTIVE")
                .build();

        // when
        trackerKafkaProducer.sendHistory(dto);

        // then
        verify(historyNewTemplate).send(historyRecordCaptor.capture());
        
        ProducerRecord<String, HistoryNewDto> record = historyRecordCaptor.getValue();
        assertThat(record.key()).isEqualTo(expectedUuid.toString());
    }

    @Test
    void sendAdditional_ShouldSendPlannedDateToKafka() {
        // given
        UUID entityUuid = UUID.randomUUID();
        PlannedDateDto dto = PlannedDateDto.builder()
                .entityUuid(entityUuid)
                .plannedDate("2024-12-31")
                .build();

        // when
        trackerKafkaProducer.sendAdditional(dto);

        // then
        verify(plannedDateTemplate).send(plannedDateRecordCaptor.capture());
        
        ProducerRecord<String, PlannedDateDto> record = plannedDateRecordCaptor.getValue();
        
        assertThat(record.topic()).isEqualTo(ADDITIONAL_TOPIC);
        assertThat(record.key()).isEqualTo(entityUuid.toString());
        assertThat(record.value()).isSameAs(dto);
    }

    @Test
    void sendAdditional_WhenDtoHasEntityUuid_ShouldUseUuidAsKey() {
        // given
        UUID expectedUuid = UUID.randomUUID();
        PlannedDateDto dto = PlannedDateDto.builder()
                .entityUuid(expectedUuid)
                .build();

        // when
        trackerKafkaProducer.sendAdditional(dto);

        // then
        verify(plannedDateTemplate).send(plannedDateRecordCaptor.capture());
        
        ProducerRecord<String, PlannedDateDto> record = plannedDateRecordCaptor.getValue();
        assertThat(record.key()).isEqualTo(expectedUuid.toString());
    }
}
```
