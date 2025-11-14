```java




@ExtendWith(MockitoExtension.class)
class HelperSupplierCriteriaBuilderTest {

    @Mock
    private SearchRequestProperties properties;

    @InjectMocks
    private HelperSupplierCriteriaBuilder builder;

    // --------------------------------------------------------------------
    // buildCriteria
    // --------------------------------------------------------------------

    @Test
    void givenInnAndKpp_whenBuildCriteria_thenReturnMapWithBothAttributes() {
        // given
        String inn = "7700000000";
        String kpp = "770001001";

        when(properties.getAttributeIdForInn()).thenReturn("innAttr");
        when(properties.getAttributeIdForKpp()).thenReturn("kppAttr");

        // when
        Map<String, List<String>> result = builder.buildCriteria(inn, kpp);

        // then
        assertEquals(
                Map.of(
                        "innAttr", List.of(inn),
                        "kppAttr", List.of(kpp)
                ),
                result
        );

        verify(properties).getAttributeIdForInn();
        verify(properties).getAttributeIdForKpp();
    }

    @Test
    void givenOnlyInn_whenBuildCriteria_thenReturnMapWithInnOnly() {
        // given
        String inn = "7700000000";
        String kpp = null;

        when(properties.getAttributeIdForInn()).thenReturn("innAttr");

        // when
        Map<String, List<String>> result = builder.buildCriteria(inn, kpp);

        // then
        assertEquals(
                Map.of("innAttr", List.of(inn)),
                result
        );

        verify(properties).getAttributeIdForInn();
        verify(properties, never()).getAttributeIdForKpp();
    }

    @Test
    void givenNullOrBlankInnAndKpp_whenBuildCriteria_thenReturnEmptyMapAndDoNotUseProperties() {
        // given
        String inn = "   ";   // isBlank() == true
        String kpp = null;

        // when
        Map<String, List<String>> result = builder.buildCriteria(inn, kpp);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(properties);
    }

    // --------------------------------------------------------------------
    // buildInnKppKey
    // --------------------------------------------------------------------

    @Test
    void givenInnAndKpp_whenBuildInnKppKey_thenReturnFormattedKey() {
        // given
        String inn = "7700000000";
        String kpp = "770001001";

        // when
        String key = builder.buildInnKppKey(inn, kpp);

        // then
        assertEquals("inn:7700000000:kpp:770001001", key);
        verifyNoInteractions(properties);
    }

    @Test
    void givenNullValues_whenBuildInnKppKey_thenNullsAreIncludedInFormat() {
        // given
        String inn = null;
        String kpp = null;

        // when
        String key = builder.buildInnKppKey(inn, kpp);

        // then
        assertEquals("inn:null:kpp:null", key);
        verifyNoInteractions(properties);
    }

    // --------------------------------------------------------------------
    // buildCriteriaFromKey
    // --------------------------------------------------------------------

    @Test
    void givenValidKey_whenBuildCriteriaFromKey_thenReturnCriteriaForInnAndKpp() {
        // given
        String key = "inn:7700000000:kpp:770001001";

        when(properties.getAttributeIdForInn()).thenReturn("innAttr");
        when(properties.getAttributeIdForKpp()).thenReturn("kppAttr");

        // when
        Map<String, List<String>> result = builder.buildCriteriaFromKey(key);

        // then
        assertEquals(
                Map.of(
                        "innAttr", List.of("7700000000"),
                        "kppAttr", List.of("770001001")
                ),
                result
        );

        // под капотом вызывается buildCriteria(inn, kpp), поэтому оба метода дергаются
        verify(properties).getAttributeIdForInn();
        verify(properties).getAttributeIdForKpp();
    }

    @Test
    void givenBlankKey_whenBuildCriteriaFromKey_thenReturnEmptyMapAndDoNotUseProperties() {
        // given
        String key = "   ";

        // when
        Map<String, List<String>> result = builder.buildCriteriaFromKey(key);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(properties);
    }

    @Test
    void givenMalformedKey_whenBuildCriteriaFromKey_thenReturnEmptyMapAndDoNotUseProperties() {
        // given
        String key = "inn:7700000000"; // parts.length != 4

        // when
        Map<String, List<String>> result = builder.buildCriteriaFromKey(key);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(properties);
    }
}
```
