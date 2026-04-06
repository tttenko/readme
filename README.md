```java
@ExtendWith(MockitoExtension.class)
class StsTrackerSchemeConfigTest {

    @Mock
    private PathSchemeFactory pathSchemeFactory;

    @Mock
    private Scheme scheme;

    @InjectMocks
    private StsTrackerSchemeConfig stsTrackerSchemeConfig;

    @BeforeEach
    void setUp() {
        ReflectionTestUtils.setField(stsTrackerSchemeConfig, "schemePath", "status-sts.yml");
    }

    @Test
    void givenSchemePath_whenStsTrackerScheme_thenReturnCreatedScheme() {
        // given
        when(pathSchemeFactory.createInstance("status-sts.yml"))
                .thenReturn(scheme);

        // when
        Scheme actualScheme = stsTrackerSchemeConfig.stsTrackerScheme();

        // then
        assertThat(actualScheme).isSameAs(scheme);

        verify(pathSchemeFactory).createInstance("status-sts.yml");
        verifyNoMoreInteractions(pathSchemeFactory);
    }
}
```
