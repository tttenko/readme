```java

@ExtendWith(MockitoExtension.class)
class CurrencyRateKafkaListenerTestIT {

    private static final String TOPIC = "topic";
    private static final int PARTITION = 0;
    private static final long OFFSET = 1L;

    @Mock
    private XmlKafkaRecordProcessor recordProcessor;

    @Mock
    private CurrencyRateRoutesRegistry routesRegistry;

    @InjectMocks
    private CurrencyRateKafkaListener listener;

    @Test
    void givenMessage_whenHandleMessage_thenDelegatesToProcessor() throws Exception {
        // GIVEN
        ConsumerRecord<String, String> record = new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, "k", "<PutEODPriceNf/>");

        // WHEN
        listener.handleMessage(record);

        // THEN
        verify(recordProcessor).process(record, routesRegistry);
        verifyNoMoreInteractions(recordProcessor);
    }

    @Test
    void givenProcessorThrows_whenHandleMessage_thenPropagatesException() throws Exception {
        // GIVEN
        ConsumerRecord<String, String> record = new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, "k", "<PutEODPriceNf/>");
        Exception boom = new Exception("boom");
        doThrow(boom).when(recordProcessor).process(record, routesRegistry);

        // WHEN
        Throwable thrown = catchThrowable(() -> listener.handleMessage(record));

        // THEN
        assertThat(thrown).isSameAs(boom);
        verify(recordProcessor).process(record, routesRegistry);
        verifyNoMoreInteractions(recordProcessor);
    }
}


@ExtendWith(MockitoExtension.class)
class CurrencyRateKeyResolverTestIT {

    private static final String TOPIC = "topic";
    private static final int PARTITION = 3;
    private static final long OFFSET = 10L;

    @Mock
    private FxRateXmlSupport fxRateXmlSupport;

    @InjectMocks
    private CurrencyRateKeyResolver resolver;

    @Test
    void givenValidXml_whenResolve_thenReturnsRootTag() {
        // GIVEN
        String xml = "<PutEODPriceNf/>";
        ConsumerRecord<String, String> record = new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, "k", xml);

        when(fxRateXmlSupport.extractRoot(xml)).thenReturn("PutEODPriceNf");

        // WHEN
        String routeKey = resolver.resolve(record);

        // THEN
        assertThat(routeKey).isEqualTo("PutEODPriceNf");
        verify(fxRateXmlSupport).extractRoot(xml);
    }

    @Test
    void givenExtractRootThrows_whenResolve_thenWrapsWithMetadata() {
        // GIVEN
        String xml = "<broken>";
        ConsumerRecord<String, String> record = new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, "k", xml);

        IllegalArgumentException boom = new IllegalArgumentException("bad xml");
        when(fxRateXmlSupport.extractRoot(xml)).thenThrow(boom);

        // WHEN
        Throwable thrown = catchThrowable(() -> resolver.resolve(record));

        // THEN
        assertThat(thrown)
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Cannot resolve routeKey from XML root tag.")
                .hasMessageContaining("topic=" + TOPIC)
                .hasMessageContaining("partition=" + PARTITION)
                .hasMessageContaining("offset=" + OFFSET)
                .hasCause(boom);
    }

    @Test
    void givenNullRecord_whenResolve_thenThrowsNullPointerException() {
        // GIVEN
        ConsumerRecord<String, String> record = null;

        // WHEN
        Throwable thrown = catchThrowable(() -> resolver.resolve(record));

        // THEN
        assertThat(thrown).isInstanceOf(NullPointerException.class);
    }
}


@ExtendWith(MockitoExtension.class)
class XmlKafkaRecordProcessorTestIT {

    private static final String TOPIC = "topic";
    private static final int PARTITION = 0;
    private static final long OFFSET = 1L;

    private static final String ROUTE_KEY = "PutEODPriceNf";
    private static final String KAFKA_KEY = "kafkaKey";

    private static final String XML_RAW = "\uFEFF<PutEODPriceNf></PutEODPriceNf>"; // с BOM
    private static final String XML_STRIPPED = "<PutEODPriceNf></PutEODPriceNf>";

    private static final class DummyDto {}

    @Mock private XmlMapper xmlMapper;
    @Mock private CurrencyRateKeyResolver keyResolver;
    @Mock private FxRateXmlSupport fxRateXmlSupport;
    @Mock private RoutesRegistry registry;

    @Mock private Consumer<DummyDto> consumer;

    private XmlKafkaRecordProcessor processor;

    private ConsumerRecord<String, String> record;

    @BeforeEach
    void setUp() {
        processor = new XmlKafkaRecordProcessor(xmlMapper, keyResolver, fxRateXmlSupport);
        record = new ConsumerRecord<>(TOPIC, PARTITION, OFFSET, KAFKA_KEY, XML_RAW);
    }

    @Test
    void givenValidRecord_whenProcess_thenResolvesRouteParsesXmlAndDispatches() throws Exception {
        // GIVEN
        DummyDto dto = new DummyDto();
        RouteDefinition<DummyDto> route = RouteDefinition.route(ROUTE_KEY, DummyDto.class, consumer);

        when(keyResolver.resolve(record)).thenReturn(ROUTE_KEY);
        when(registry.<DummyDto>getRequired(ROUTE_KEY)).thenReturn(route);
        when(fxRateXmlSupport.stripBom(XML_RAW)).thenReturn(XML_STRIPPED);
        when(xmlMapper.readValue(XML_STRIPPED, DummyDto.class)).thenReturn(dto);

        // WHEN
        processor.process(record, registry);

        // THEN
        verify(keyResolver).resolve(record);
        verify(registry).getRequired(ROUTE_KEY);
        verify(fxRateXmlSupport).stripBom(XML_RAW);
        verify(xmlMapper).readValue(XML_STRIPPED, DummyDto.class);
        verify(consumer).accept(dto);
        verifyNoMoreInteractions(consumer);
    }

    @Test
    void givenXmlMapperThrows_whenProcess_thenThrowsIllegalArgumentExceptionAndDoesNotDispatch() throws Exception {
        // GIVEN
        RouteDefinition<DummyDto> route = RouteDefinition.route(ROUTE_KEY, DummyDto.class, consumer);

        when(keyResolver.resolve(record)).thenReturn(ROUTE_KEY);
        when(registry.<DummyDto>getRequired(ROUTE_KEY)).thenReturn(route);
        when(fxRateXmlSupport.stripBom(XML_RAW)).thenReturn(XML_STRIPPED);

        RuntimeException boom = new RuntimeException("boom");
        when(xmlMapper.readValue(XML_STRIPPED, DummyDto.class)).thenThrow(boom);

        // WHEN
        Throwable thrown = catchThrowable(() -> processor.process(record, registry));

        // THEN
        assertThat(thrown)
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Не удалось распарсить XML")
                .hasMessageContaining("routeKey=" + ROUTE_KEY)
                .hasMessageContaining("record.key=" + KAFKA_KEY)
                .hasCause(boom);

        verify(consumer, never()).accept(any());
    }

    @Test
    void givenKeyResolverThrows_whenProcess_thenPropagatesAndDoesNotTouchMapper() {
        // GIVEN
        IllegalArgumentException boom = new IllegalArgumentException("bad root");
        when(keyResolver.resolve(record)).thenThrow(boom);

        // WHEN
        Throwable thrown = catchThrowable(() -> processor.process(record, registry));

        // THEN
        assertThat(thrown).isSameAs(boom);

        verify(keyResolver).resolve(record);
        verifyNoInteractions(registry);
        verifyNoInteractions(fxRateXmlSupport);
        verifyNoInteractions(xmlMapper);
        verifyNoInteractions(consumer);
    }

    @Test
    void givenRegistryReturnsNullRoute_whenProcess_thenThrowsNullPointerException() {
        // GIVEN
        when(keyResolver.resolve(record)).thenReturn(ROUTE_KEY);
        when(registry.getRequired(ROUTE_KEY)).thenReturn(null);

        // WHEN
        Throwable thrown = catchThrowable(() -> processor.process(record, registry));

        // THEN
        assertThat(thrown).isInstanceOf(NullPointerException.class);
        verifyNoInteractions(fxRateXmlSupport);
        verifyNoInteractions(xmlMapper);
        verifyNoInteractions(consumer);
    }

    @Test
    void givenNullRecord_whenProcess_thenThrowsNullPointerException() {
        // GIVEN
        ConsumerRecord<String, String> r = null;

        // WHEN
        Throwable thrown = catchThrowable(() -> processor.process(r, registry));

        // THEN
        assertThat(thrown).isInstanceOf(NullPointerException.class);
    }

    @Test
    void givenNullRegistry_whenProcess_thenThrowsNullPointerException() {
        // GIVEN
        RoutesRegistry reg = null;

        // WHEN
        Throwable thrown = catchThrowable(() -> processor.process(record, reg));

        // THEN
        assertThat(thrown).isInstanceOf(NullPointerException.class);
    }
}


@ExtendWith(MockitoExtension.class)
class CurrencyRateIngestRoutesProviderTestIT {

    private static final String KEY = "PutEODPriceNf";

    @Mock
    private FxRateIngestService ingestService;

    @Mock
    private PutEodPriceNfDto dto;

    @InjectMocks
    private CurrencyRateIngestRoutesProvider provider;

    private List<RouteDefinition<?>> routes;

    @BeforeEach
    void setUp() {
        routes = provider.routes();
    }

    @Test
    void givenProvider_whenRoutes_thenReturnsSingleRouteWithExpectedKeyAndDtoClass() {
        // GIVEN/WHEN
        RouteDefinition<?> route = routes.get(0);

        // THEN
        assertThat(routes).hasSize(1);
        assertThat(route.getKey()).isEqualTo(KEY);
        assertThat(route.getDtoClass()).isEqualTo(PutEodPriceNfDto.class);
        assertThat(route.getConsumer()).isNotNull();
    }

    @Test
    @SuppressWarnings("unchecked")
    void givenRoute_whenConsumerAcceptsDto_thenIngestServiceCalled() {
        // GIVEN
        RouteDefinition<PutEodPriceNfDto> route = (RouteDefinition<PutEodPriceNfDto>) routes.get(0);

        // WHEN
        route.getConsumer().accept(dto);

        // THEN
        verify(ingestService).ingest(dto);
        verifyNoMoreInteractions(ingestService);
    }
}


@ExtendWith(MockitoExtension.class)
class CurrencyRateRoutesRegistryTestIT {

    private static final String KEY = "PutEODPriceNf";

    private static final class DummyDto {}

    @Mock
    private RateKafkaRoutesProvider provider;

    @Test
    void givenNullProviders_whenConstruct_thenThrowsNullPointerException() {
        // GIVEN
        List<RateKafkaRoutesProvider> providers = null;

        // WHEN
        Throwable thrown = catchThrowable(() -> new CurrencyRateRoutesRegistry(providers));

        // THEN
        assertThat(thrown).isInstanceOf(NullPointerException.class);
    }

    @Test
    void givenProviders_whenConstruct_thenResolvesRoute() {
        // GIVEN
        when(provider.routes()).thenReturn(List.of(
                RouteDefinition.route(KEY, DummyDto.class, dto -> {})
        ));

        // WHEN
        CurrencyRateRoutesRegistry registry = new CurrencyRateRoutesRegistry(List.of(provider));

        // THEN
        assertThat(registry.getRequired(KEY)).isNotNull();
        assertThat(registry.getRequired(KEY).getKey()).isEqualTo(KEY);
        assertThat(registry.getRequired(KEY).getDtoClass()).isEqualTo(DummyDto.class);
    }
}

```
