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
    private ArgumentCaptor<String> historyTopicCaptor;

    @Captor
    private ArgumentCaptor<String> historyKeyCaptor;

    @Captor
    private ArgumentCaptor<HistoryNewDto> historyDtoCaptor;

    @Captor
    private ArgumentCaptor<String> additionalTopicCaptor;

    @Captor
    private ArgumentCaptor<String> additionalKeyCaptor;

    @Captor
    private ArgumentCaptor<PlannedDateDto> additionalDtoCaptor;

    private static final String HISTORY_TOPIC = "tracker_history";
    private static final String ADDITIONAL_TOPIC = "tracker_additional";

    @BeforeEach
    void setUp() {
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
        verify(historyNewTemplate).send(
                historyTopicCaptor.capture(),
                historyKeyCaptor.capture(),
                historyDtoCaptor.capture()
        );
        
        assertThat(historyTopicCaptor.getValue()).isEqualTo(HISTORY_TOPIC);
        assertThat(historyKeyCaptor.getValue()).isEqualTo(entityUuid.toString());
        assertThat(historyDtoCaptor.getValue()).isSameAs(dto);
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
        verify(historyNewTemplate).send(
                historyTopicCaptor.capture(),
                historyKeyCaptor.capture(),
                historyDtoCaptor.capture()
        );
        
        assertThat(historyKeyCaptor.getValue()).isEqualTo(expectedUuid.toString());
    }

    @Test
    void sendAdditional_ShouldSendPlannedDateToKafka() {
        // given
        UUID entityUuid = UUID.randomUUID();
        ZonedDateTime plannedDate = ZonedDateTime.now();
        PlannedDateDto dto = PlannedDateDto.builder()
                .entityUuid(entityUuid)
                .plannedDate(plannedDate)
                .build();

        // when
        trackerKafkaProducer.sendAdditional(dto);

        // then
        verify(plannedDateTemplate).send(
                additionalTopicCaptor.capture(),
                additionalKeyCaptor.capture(),
                additionalDtoCaptor.capture()
        );
        
        assertThat(additionalTopicCaptor.getValue()).isEqualTo(ADDITIONAL_TOPIC);
        assertThat(additionalKeyCaptor.getValue()).isEqualTo(entityUuid.toString());
        assertThat(additionalDtoCaptor.getValue()).isSameAs(dto);
    }

    @Test
    void sendAdditional_WhenDtoHasEntityUuid_ShouldUseUuidAsKey() {
        // given
        UUID expectedUuid = UUID.randomUUID();
        ZonedDateTime plannedDate = ZonedDateTime.now();
        PlannedDateDto dto = PlannedDateDto.builder()
                .entityUuid(expectedUuid)
                .plannedDate(plannedDate)
                .build();

        // when
        trackerKafkaProducer.sendAdditional(dto);

        // then
        verify(plannedDateTemplate).send(
                additionalTopicCaptor.capture(),
                additionalKeyCaptor.capture(),
                additionalDtoCaptor.capture()
        );
        
        assertThat(additionalKeyCaptor.getValue()).isEqualTo(expectedUuid.toString());
    }
}
```
