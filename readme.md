```java/**
@ExtendWith(MockitoExtension.class)
class CurrencyRateKafkaListenerTest {

    @Mock
    private CurrencyRateMessageProcessor processor;

    private CurrencyRateKafkaListener listener;

    @BeforeEach
    void setUp() {
        listener = new CurrencyRateKafkaListener(processor);
    }

    @Test
    void handleMessage_shouldDelegateToProcessor() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 0, 1L, "k", "<PutEODPriceNf/>");

        listener.handleMessage(record);

        verify(processor).process(record);
        verifyNoMoreInteractions(processor);
    }

    @Test
    void handleMessage_shouldPropagateExceptionFromProcessor() {
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 0, 1L, "k", "<bad/>");
        RuntimeException ex = new RuntimeException("boom");

        doThrow(ex).when(processor).process(record);

        Throwable thrown = catchThrowable(() -> listener.handleMessage(record));

        assertThat(thrown).isSameAs(ex);
        verify(processor).process(record);
        verifyNoMoreInteractions(processor);
    }
}

```
