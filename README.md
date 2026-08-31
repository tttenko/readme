```java

class JiraInitiativeOrganizationResolverTest {

    private lateinit var resolver: JiraInitiativeOrganizationResolver

    @BeforeEach
    fun setUp() {
        resolver = JiraInitiativeOrganizationResolver()
    }

    // --- Priority 1: executorUnits -> division ---

    @Test
    fun `resolveOrganization should find division contained in executor unit`() {
        val block = createBlock(label = "Corporate")
        val division = createDivision(
            label = "Corporate_Lending",
            block = block,
        )

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "Corporate_Lending" to division,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("Unknown"),
            executorUnits = listOf("КИБ/Corporate_Lending"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(division)
        assertThat(result.block).isSameAs(block)
    }

    @Test
    fun `resolveOrganization should try all executor unit values for division match`() {
        val division = createDivision(
            label = "SecondMatch",
            block = createBlock(label = "BlockX"),
        )

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "FirstMatch" to createDivision(label = "FirstMatch"),
                "SecondMatch" to division,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = listOf(
                "NoMatch",
                "Prefix/SecondMatch/Suffix",
            ),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(division)
    }

    @Test
    fun `resolveOrganization should match division ignoring case`() {
        val division = createDivision(label = "Corporate_Lending")

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "Corporate_Lending" to division,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = listOf("КИБ/CORPORATE_LENDING"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(division)
    }

    // --- Priority 2: initiatorUnits -> division ---

    @Test
    fun `resolveOrganization should find division contained in initiator unit as fallback`() {
        val division = createDivision(label = "InitiativeDiv")

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "InitiativeDiv" to division,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("КИБ/InitiativeDiv"),
            executorUnits = listOf("NonMatching"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(division)
    }

    // --- Priority between divisions and blocks ---

    @Test
    fun `resolveOrganization should prefer initiator division over executor block`() {
        val division = createDivision(
            label = "InitiatorDivision",
            block = createBlock(label = "ParentBlock"),
        )

        val executorBlock = createBlock(label = "ExecutorBlock")

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "InitiatorDivision" to division,
            ),
            blocks = mapOf(
                "ExecutorBlock" to executorBlock,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("КИБ/InitiatorDivision"),
            executorUnits = listOf("КИБ/ExecutorBlock"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(division)
        assertThat(result.block).isSameAs(division.block)
    }

    // --- Priority 3: executorUnits -> block ---

    @Test
    fun `resolveOrganization should find block contained in executor unit when no division matches`() {
        val block = createBlock(label = "ExecutorBlock")

        val referenceData = mockkReferenceData(
            blocks = mapOf(
                "ExecutorBlock" to block,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("NoMatch"),
            executorUnits = listOf("Prefix/ExecutorBlock"),
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isSameAs(block)
    }

    // --- Priority 4: initiatorUnits -> block ---

    @Test
    fun `resolveOrganization should find block contained in initiator unit as last resort`() {
        val block = createBlock(label = "InitiativeBlock")

        val referenceData = mockkReferenceData(
            blocks = mapOf(
                "InitiativeBlock" to block,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("Prefix/InitiativeBlock"),
            executorUnits = listOf("NoBlockMatch"),
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isSameAs(block)
    }

    // --- Most specific match ---

    @Test
    fun `resolveOrganization should choose longest matching division label`() {
        val genericDivision = createDivision(label = "Corporate")
        val specificDivision = createDivision(label = "Corporate_Lending")

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "Corporate" to genericDivision,
                "Corporate_Lending" to specificDivision,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = listOf("КИБ/Corporate_Lending"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(specificDivision)
    }

    @Test
    fun `resolveOrganization should choose longest matching block label`() {
        val genericBlock = createBlock(label = "КИБ")
        val specificBlock = createBlock(label = "КИБ/Corporate")

        val referenceData = mockkReferenceData(
            blocks = mapOf(
                "КИБ" to genericBlock,
                "КИБ/Corporate" to specificBlock,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = listOf("КИБ/Corporate/SomeDivision"),
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isSameAs(specificBlock)
    }

    // --- Priority executor -> initiator ---

    @Test
    fun `resolveOrganization should prefer executor division over initiator division`() {
        val executorDivision = createDivision(label = "ExecutorDiv")
        val initiatorDivision = createDivision(label = "InitiatorDiv")

        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "ExecutorDiv" to executorDivision,
                "InitiatorDiv" to initiatorDivision,
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("КИБ/InitiatorDiv"),
            executorUnits = listOf("КИБ/ExecutorDiv"),
            referenceData = referenceData,
        )

        assertThat(result.division).isSameAs(executorDivision)
    }

    // --- No match ---

    @Test
    fun `resolveOrganization should return nulls when nothing matches`() {
        val referenceData = mockkReferenceData(
            divisions = mapOf(
                "KnownDivision" to createDivision("KnownDivision"),
            ),
            blocks = mapOf(
                "KnownBlock" to createBlock("KnownBlock"),
            ),
        )

        val result = resolver.resolveOrganization(
            initiatorUnits = listOf("UnknownInitiator"),
            executorUnits = listOf("UnknownExecutor"),
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isNull()
    }

    @Test
    fun `resolveOrganization should return nulls for null organization fields`() {
        val referenceData = mockkReferenceData()

        val result = resolver.resolveOrganization(
            initiatorUnits = null,
            executorUnits = null,
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isNull()
    }

    @Test
    fun `resolveOrganization should return nulls for empty organization fields`() {
        val referenceData = mockkReferenceData()

        val result = resolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = emptyList(),
            referenceData = referenceData,
        )

        assertThat(result.division).isNull()
        assertThat(result.block).isNull()
    }

    private fun createDivision(
        label: String,
        block: BlockEntity? = null,
    ): DivisionEntity {
        return DivisionEntity().apply {
            this.label = label
            this.block = block
        }
    }

    private fun createBlock(
        label: String,
    ): BlockEntity {
        return BlockEntity().apply {
            this.label = label
        }
    }

    private fun mockkReferenceData(
        divisions: Map<String, DivisionEntity> = emptyMap(),
        blocks: Map<String, BlockEntity> = emptyMap(),
    ): JiraImportReferenceData {
        return JiraImportReferenceData(
            strategiesByJiraKey = emptyMap(),
            enablersByNormalizedName = emptyMap(),
            statusesByCode = emptyMap(),
            qualityGates = emptyList(),
            divisionsByLabel = divisions,
            blocksByLabel = blocks,
            initiativeTypesByCode = emptyMap(),
        )
    }
}
```
