```java/**
@ExtendWith(MockitoExtension.class)
class CurrencyRateMessageProcessorTest {

    @Mock
    private PutEodPriceNfXmlParser parser;

    @Mock
    private FxRateIngestService ingestService;

    private CurrencyRateMessageProcessor processor;

    @BeforeEach
    void setUp() {
        processor = new CurrencyRateMessageProcessor(parser, ingestService);
    }

    @Test
    void process_shouldThrow_whenRecordIsNull() {
        Throwable thrown = catchThrowable(() -> processor.process(null));

        assertThat(thrown)
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Kafka record is null");

        verifyNoInteractions(parser, ingestService);
    }

    @Test
    void process_shouldThrow_whenValueIsNull() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 1, 10L, "k", null);

        Throwable thrown = catchThrowable(() -> processor.process(record));

        assertThat(thrown)
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Kafka record value is null/blank")
            .hasMessageContaining("topic=t")
            .hasMessageContaining("partition=1")
            .hasMessageContaining("offset=10");

        verifyNoInteractions(parser, ingestService);
    }

    @Test
    void process_shouldThrow_whenValueIsBlank() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("topicA", 0, 5L, "k", "   ");

        Throwable thrown = catchThrowable(() -> processor.process(record));

        assertThat(thrown)
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Kafka record value is null/blank")
            .hasMessageContaining("topic=topicA")
            .hasMessageContaining("partition=0")
            .hasMessageContaining("offset=5");

        verifyNoInteractions(parser, ingestService);
    }

    @Test
    void process_shouldParseAndIngest_whenValueIsPresent() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 2, 99L, "k1", "<PutEODPriceNf/>");
        PutEodPriceNfDto dto = mock(PutEodPriceNfDto.class);

        when(parser.parse(record)).thenReturn(dto);

        processor.process(record);

        verify(parser).parse(record);
        verify(ingestService).ingest(dto);
        verifyNoMoreInteractions(parser, ingestService);
    }

    @Test
    void process_shouldNotCallIngest_whenParserThrows() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 2, 99L, "k1", "<bad/>");
        RuntimeException ex = new InvalidCurrencyRateXmlException("parse failed", new RuntimeException("cause"));

        when(parser.parse(record)).thenThrow(ex);

        Throwable thrown = catchThrowable(() -> processor.process(record));

        assertThat(thrown).isSameAs(ex);

        verify(parser).parse(record);
        verifyNoInteractions(ingestService);
        verifyNoMoreInteractions(parser);
    }

    @Test
    void process_shouldPropagate_whenIngestThrows() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 2, 99L, "k1", "<PutEODPriceNf/>");
        PutEodPriceNfDto dto = mock(PutEodPriceNfDto.class);
        RuntimeException ex = new RuntimeException("db down");

        when(parser.parse(record)).thenReturn(dto);
        doThrow(ex).when(ingestService).ingest(dto);

        Throwable thrown = catchThrowable(() -> processor.process(record));

        assertThat(thrown).isSameAs(ex);

        verify(parser).parse(record);
        verify(ingestService).ingest(dto);
        verifyNoMoreInteractions(parser, ingestService);
    }
}

```
