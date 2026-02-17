```java/**
@ExtendWith(MockitoExtension.class)
class PutEodPriceNfXmlParserTest {

    @Mock
    private XmlMapper xmlMapper;

    private PutEodPriceNfXmlParser parser;

    @BeforeEach
    void setUp() {
        parser = new PutEodPriceNfXmlParser(xmlMapper);
    }

    @Test
    void parse_shouldReturnDto_whenXmlIsValid() throws Exception {
        // given
        ConsumerRecord<String, String> record = new ConsumerRecord<>("topicA", 2, 10L, "k1", "<PutEODPriceNf/>");
        PutEodPriceNfDto dto = mock(PutEodPriceNfDto.class);

        when(xmlMapper.readValue("<PutEODPriceNf/>", PutEodPriceNfDto.class)).thenReturn(dto);

        // when
        PutEodPriceNfDto result = parser.parse(record);

        // then
        assertThat(result).isSameAs(dto);
        verify(xmlMapper).readValue("<PutEODPriceNf/>", PutEodPriceNfDto.class);
        verifyNoMoreInteractions(xmlMapper);
    }

    @Test
    void parse_shouldStripBomAndTrim_beforeDeserialization() throws Exception {
        // given
        String raw = "\uFEFF   <PutEODPriceNf/>   ";
        String normalized = "<PutEODPriceNf/>";

        ConsumerRecord<String, String> record = new ConsumerRecord<>("topicA", 0, 1L, "k", raw);
        PutEodPriceNfDto dto = mock(PutEodPriceNfDto.class);

        when(xmlMapper.readValue(normalized, PutEodPriceNfDto.class)).thenReturn(dto);

        // when
        PutEodPriceNfDto result = parser.parse(record);

        // then
        assertThat(result).isSameAs(dto);

        // важно: проверяем, что в readValue пришла уже нормализованная строка
        ArgumentCaptor<String> xmlCaptor = ArgumentCaptor.forClass(String.class);
        verify(xmlMapper).readValue(xmlCaptor.capture(), eq(PutEodPriceNfDto.class));
        assertThat(xmlCaptor.getValue()).isEqualTo(normalized);

        verifyNoMoreInteractions(xmlMapper);
    }

    @Test
    void parse_shouldPassEmptyStringToMapper_whenValueIsNull() throws Exception {
        // given
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 1, 2L, "k", null);
        PutEodPriceNfDto dto = mock(PutEodPriceNfDto.class);

        when(xmlMapper.readValue("", PutEodPriceNfDto.class)).thenReturn(dto);

        // when
        PutEodPriceNfDto result = parser.parse(record);

        // then
        assertThat(result).isSameAs(dto);
        verify(xmlMapper).readValue("", PutEodPriceNfDto.class);
        verifyNoMoreInteractions(xmlMapper);
    }

    @Test
    void parse_shouldWrapExceptionIntoInvalidCurrencyRateXmlException_withKafkaContext() throws Exception {
        // given
        ConsumerRecord<String, String> record = new ConsumerRecord<>("ratesTopic", 3, 42L, "myKey", "<bad/>");
        RuntimeException cause = new RuntimeException("boom");

        when(xmlMapper.readValue("<bad/>", PutEodPriceNfDto.class)).thenThrow(cause);

        // when
        Throwable thrown = catchThrowable(() -> parser.parse(record));

        // then
        assertThat(thrown)
            .isInstanceOf(InvalidCurrencyRateXmlException.class)
            .hasMessageContaining("Не удалось распарсить PutEODPriceNF XML")
            .hasMessageContaining("topic=ratesTopic")
            .hasMessageContaining("partition=3")
            .hasMessageContaining("offset=42")
            .hasMessageContaining("key=myKey")
            .hasCause(cause);

        verify(xmlMapper).readValue("<bad/>", PutEodPriceNfDto.class);
        verifyNoMoreInteractions(xmlMapper);
    }

    @Test
    void parse_shouldThrowNullPointerException_whenRecordIsNull() {
        // given / when
        Throwable thrown = catchThrowable(() -> parser.parse(null));

        // then
        assertThat(thrown).isInstanceOf(NullPointerException.class);
        verifyNoInteractions(xmlMapper);
    }

    @Test
    void parse_shouldIncludeKeyNullInErrorMessage_whenKeyIsNull() throws Exception {
        // given
        ConsumerRecord<String, String> record = new ConsumerRecord<>("t", 0, 5L, null, "<bad/>");
        Exception cause = new IllegalArgumentException("invalid");

        when(xmlMapper.readValue("<bad/>", PutEodPriceNfDto.class)).thenThrow(cause);

        // when
        Throwable thrown = catchThrowable(() -> parser.parse(record));

        // then
        assertThat(thrown)
            .isInstanceOf(InvalidCurrencyRateXmlException.class)
            .hasMessageContaining("topic=t")
            .hasMessageContaining("partition=0")
            .hasMessageContaining("offset=5")
            .hasMessageContaining("key=null")
            .hasCause(cause);
    }
}


```
