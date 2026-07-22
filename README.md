```java

@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsResponseBuilderTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var valueCalculator:
        InitiativeMetricPreAnalyticsValueCalculator

    private lateinit var responseBuilder:
        InitiativeMetricPreAnalyticsResponseBuilder

    private lateinit var metric:
        MetricsDirectoryEntity

    private lateinit var autonomousMetricType:
        InitiativeMetricTypeEntity

    private lateinit var copilotMetricType:
        InitiativeMetricTypeEntity

    @BeforeEach
    fun setUp() {
        every {
            messageProvider[PRE_ANALYTICS_CODE_NOT_CONFIGURED]
        } returns PRE_ANALYTICS_CODE_NOT_CONFIGURED_MESSAGE

        responseBuilder =
            InitiativeMetricPreAnalyticsResponseBuilder(
                messageProvider = messageProvider,
                valueCalculator = valueCalculator,
            )

        metric =
            mockk {
                every { id } returns METRIC_ID
                every { code } returns METRIC_CODE
                every { name } returns METRIC_NAME
                every { unit } returns METRIC_UNIT
                every { direction } returns MORE_IS_BETTER
            }

        autonomousMetricType =
            metricType(
                agentType = AUTONOMOUS,
            )

        copilotMetricType =
            metricType(
                agentType = COPILOT,
            )
    }

    @Test
    fun shouldReturnEmptyListWhenRequestedAgentTypeDoesNotExist() {
        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS,
                metricTypes = listOf(copilotMetricType),
                metrics = listOf(metric),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).isEmpty()

        verify {
            valueCalculator wasNot Called
        }
    }

    @Test
    fun shouldBuildEmptyResponseWhenPreviousValueIsAbsent() {
        val response =
            buildResponse(
                valuesByKey = emptyMap(),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )

        verify {
            valueCalculator wasNot Called
        }
    }

    @Test
    fun shouldBuildEmptyResponseWhenPreviousMetricValueIsNull() {
        val previousValue =
            metricValue(
                metricValue = null,
                targetValue = TARGET_VALUE,
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )

        verify {
            valueCalculator wasNot Called
        }
    }

    @Test
    fun shouldBuildResponseWithDeltaAndDiagramValues() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns DIAGRAM_VALUE

        val previousValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )

        val beforePreviousValue =
            metricValue(
                metricValue = BEFORE_PREVIOUS_VALUE,
                targetValue = TARGET_VALUE,
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = BEFORE_PREVIOUS_MONTH,
                        ) to beforePreviousValue,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                InitiativeMetricPreAnalyticsItemResponse(
                    metricId = METRIC_ID,
                    code = METRIC_CODE,
                    name = METRIC_NAME,
                    unit = METRIC_UNIT,
                    value = SUBMITTED_VALUE,
                    targetValue = TARGET_VALUE,
                    deltaValue = DELTA_VALUE,
                    diagramValue = DIAGRAM_VALUE,
                    agentType = AUTONOMOUS,
                ),
            )

        verify(exactly = 1) {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        }

        verify(exactly = 1) {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        }
    }

    @Test
    fun shouldAddDiagramValueReturnedByCalculator() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns DIAGRAM_VALUE

        val response =
            buildResponseWithSubmittedValues()

        assertThat(response.diagramValue)
            .isEqualByComparingTo(DIAGRAM_VALUE)

        verify(exactly = 1) {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        }
    }

    @Test
    fun shouldReturnNullDiagramValueWhenCalculatorReturnsNull() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns null

        val response =
            buildResponseWithSubmittedValues()

        assertThat(response.value)
            .isEqualByComparingTo(SUBMITTED_VALUE)

        assertThat(response.targetValue)
            .isEqualByComparingTo(TARGET_VALUE)

        assertThat(response.deltaValue)
            .isEqualByComparingTo(DELTA_VALUE)

        assertThat(response.diagramValue)
            .isNull()
    }

    @Test
    fun shouldPassNullTargetValueToDiagramCalculator() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = null,
            )
        } returns null

        val previousValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = null,
            )

        val beforePreviousValue =
            metricValue(
                metricValue = BEFORE_PREVIOUS_VALUE,
                targetValue = null,
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = BEFORE_PREVIOUS_MONTH,
                        ) to beforePreviousValue,
                    ),
            )

        assertThat(response.value)
            .isEqualByComparingTo(SUBMITTED_VALUE)

        assertThat(response.targetValue)
            .isNull()

        assertThat(response.deltaValue)
            .isEqualByComparingTo(DELTA_VALUE)

        assertThat(response.diagramValue)
            .isNull()

        verify(exactly = 1) {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = null,
            )
        }
    }

    @Test
    fun shouldPassNullBeforePreviousValueToDeltaCalculator() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = null,
            )
        } returns null

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns DIAGRAM_VALUE

        val previousValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            agentType = AUTONOMOUS,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
            )

        assertThat(response.value)
            .isEqualByComparingTo(SUBMITTED_VALUE)

        assertThat(response.targetValue)
            .isEqualByComparingTo(TARGET_VALUE)

        assertThat(response.deltaValue)
            .isNull()

        assertThat(response.diagramValue)
            .isEqualByComparingTo(DIAGRAM_VALUE)

        verify(exactly = 1) {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = null,
            )
        }

        verify(exactly = 1) {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        }
    }

    @Test
    fun shouldIgnoreValuesFromAnotherAgentType() {
        val copilotValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )

        val response =
            buildResponse(
                agentType = AUTONOMOUS,
                metricTypes = listOf(autonomousMetricType),
                valuesByKey =
                    mapOf(
                        key(
                            agentType = COPILOT,
                            periodMonth = PREVIOUS_MONTH,
                        ) to copilotValue,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )

        verify {
            valueCalculator wasNot Called
        }
    }

    @Test
    fun shouldBuildCopilotResponseWithCopilotAgentType() {
        every {
            valueCalculator.calculateDeltaValue(
                direction = MORE_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = MORE_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns DIAGRAM_VALUE

        val previousValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )

        val beforePreviousValue =
            metricValue(
                metricValue = BEFORE_PREVIOUS_VALUE,
                targetValue = TARGET_VALUE,
            )

        val response =
            buildResponse(
                agentType = COPILOT,
                metricTypes = listOf(copilotMetricType),
                valuesByKey =
                    mapOf(
                        key(
                            agentType = COPILOT,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                        key(
                            agentType = COPILOT,
                            periodMonth = BEFORE_PREVIOUS_MONTH,
                        ) to beforePreviousValue,
                    ),
            )

        assertThat(response.agentType)
            .isEqualTo(COPILOT)

        assertThat(response.value)
            .isEqualByComparingTo(SUBMITTED_VALUE)

        assertThat(response.deltaValue)
            .isEqualByComparingTo(DELTA_VALUE)

        assertThat(response.diagramValue)
            .isEqualByComparingTo(DIAGRAM_VALUE)
    }

    @Test
    fun shouldPassMetricDirectionToCalculators() {
        every { metric.direction } returns LESS_IS_BETTER

        every {
            valueCalculator.calculateDeltaValue(
                direction = LESS_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        } returns DELTA_VALUE

        every {
            valueCalculator.calculateDiagramValue(
                direction = LESS_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        } returns DIAGRAM_VALUE

        val response =
            buildResponseWithSubmittedValues()

        assertThat(response.deltaValue)
            .isEqualByComparingTo(DELTA_VALUE)

        assertThat(response.diagramValue)
            .isEqualByComparingTo(DIAGRAM_VALUE)

        verify(exactly = 1) {
            valueCalculator.calculateDeltaValue(
                direction = LESS_IS_BETTER,
                submittedMetricValue = SUBMITTED_VALUE,
                beforePreviousMetricValue = BEFORE_PREVIOUS_VALUE,
            )
        }

        verify(exactly = 1) {
            valueCalculator.calculateDiagramValue(
                direction = LESS_IS_BETTER,
                actualValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )
        }
    }

    @Test
    fun shouldThrowExceptionWhenPreAnalyticsMetricCodeIsNull() {
        every { metric.code } returns null

        assertThatThrownBy {
            buildResponse(
                valuesByKey = emptyMap(),
            )
        }
            .isInstanceOf(
                AiInternalServerException::class.java,
            )
            .hasMessage(
                "Code is not configured for " +
                    "pre-analytics metric $METRIC_ID",
            )

        verify {
            valueCalculator wasNot Called
        }
    }

    @Test
    fun shouldThrowExceptionWhenPreAnalyticsMetricCodeIsBlank() {
        every { metric.code } returns " "

        assertThatThrownBy {
            buildResponse(
                valuesByKey = emptyMap(),
            )
        }
            .isInstanceOf(
                AiInternalServerException::class.java,
            )
            .hasMessage(
                "Code is not configured for " +
                    "pre-analytics metric $METRIC_ID",
            )

        verify {
            valueCalculator wasNot Called
        }
    }

    private fun buildResponseWithSubmittedValues():
        InitiativeMetricPreAnalyticsItemResponse {
        val previousValue =
            metricValue(
                metricValue = SUBMITTED_VALUE,
                targetValue = TARGET_VALUE,
            )

        val beforePreviousValue =
            metricValue(
                metricValue = BEFORE_PREVIOUS_VALUE,
                targetValue = TARGET_VALUE,
            )

        return buildResponse(
            valuesByKey =
                mapOf(
                    key(
                        agentType = AUTONOMOUS,
                        periodMonth = PREVIOUS_MONTH,
                    ) to previousValue,
                    key(
                        agentType = AUTONOMOUS,
                        periodMonth = BEFORE_PREVIOUS_MONTH,
                    ) to beforePreviousValue,
                ),
        )
    }

    private fun buildResponse(
        agentType: String = AUTONOMOUS,
        metricTypes: List<InitiativeMetricTypeEntity> =
            listOf(autonomousMetricType),
        valuesByKey:
            Map<
                InitiativeMetricPreAnalyticsResponseBuilder.MetricValueKey,
                InitiativeMetricValueEntity,
            >,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return responseBuilder
            .buildMetricsForAgentType(
                agentType = agentType,
                metricTypes = metricTypes,
                metrics = listOf(metric),
                valuesByKey = valuesByKey,
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )
            .single()
    }

    private fun emptyMetricResponse():
        InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = METRIC_ID,
            code = METRIC_CODE,
            name = METRIC_NAME,
            unit = METRIC_UNIT,
            value = null,
            targetValue = null,
            deltaValue = null,
            diagramValue = null,
            agentType = AUTONOMOUS,
        )
    }

    private fun key(
        agentType: String,
        periodMonth: YearMonth,
    ): InitiativeMetricPreAnalyticsResponseBuilder.MetricValueKey {
        return InitiativeMetricPreAnalyticsResponseBuilder
            .MetricValueKey(
                agentType = agentType,
                metricId = METRIC_ID,
                periodMonth = periodMonth,
            )
    }

    private fun metricType(
        agentType: String,
    ): InitiativeMetricTypeEntity {
        return mockk {
            every {
                this@mockk.agentType
            } returns agentType
        }
    }

    private fun metricValue(
        metricValue: BigDecimal?,
        targetValue: BigDecimal?,
    ): InitiativeMetricValueEntity {
        return mockk {
            every {
                this@mockk.metricValue
            } returns metricValue

            every {
                this@mockk.targetValue
            } returns targetValue
        }
    }

    private companion object {

        val METRIC_ID: UUID =
            UUID.fromString(
                "a860b390-b739-48e6-a694-e96582eb4e95",
            )

        val PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 6)

        val BEFORE_PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 5)

        val SUBMITTED_VALUE: BigDecimal =
            BigDecimal("120")

        val BEFORE_PREVIOUS_VALUE: BigDecimal =
            BigDecimal("100")

        val TARGET_VALUE: BigDecimal =
            BigDecimal("150")

        val DELTA_VALUE: BigDecimal =
            BigDecimal("20")

        val DIAGRAM_VALUE: BigDecimal =
            BigDecimal("80")

        const val METRIC_CODE =
            "accuracy"

        const val METRIC_NAME =
            "Полное название метрики точности"

        const val METRIC_UNIT =
            "percent"

        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"

        const val AUTONOMOUS =
            "autonomous"

        const val COPILOT =
            "copilot"

        const val PRE_ANALYTICS_CODE_NOT_CONFIGURED_MESSAGE =
            "Code is not configured for " +
                "pre-analytics metric {0}"
    }
}

@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsValueCalculatorTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    private lateinit var calculator:
        InitiativeMetricPreAnalyticsValueCalculator

    @BeforeEach
    fun setUp() {
        every {
            messageProvider[UNSUPPORTED_METRIC_DIRECTION]
        } returns UNSUPPORTED_DIRECTION_MESSAGE

        calculator =
            InitiativeMetricPreAnalyticsValueCalculator(
                messageProvider = messageProvider,
            )
    }

    @Nested
    inner class CalculateDeltaValueTest {

        @Test
        fun `should return positive delta when more is better and value increased`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("20")
        }

        @Test
        fun `should return negative delta when more is better and value decreased`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("80"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("-20")
        }

        @Test
        fun `should return positive delta when less is better and value decreased`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = LESS_IS_BETTER,
                    submittedMetricValue = BigDecimal("80"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("20")
        }

        @Test
        fun `should return negative delta when less is better and value increased`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = LESS_IS_BETTER,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("-20")
        }

        @Test
        fun `should return zero delta when more is better and values are equal`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("100"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should return zero delta when less is better and values are equal`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = LESS_IS_BETTER,
                    submittedMetricValue = BigDecimal("100"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should round delta mathematically to integer`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("102"),
                    beforePreviousMetricValue = BigDecimal("101"),
                )

            /*
             * (102 - 101) / 101 * 100 = 0.990...
             * После HALF_UP округляется до 1.
             */
            assertThat(result)
                .isEqualByComparingTo("1")
        }

        @Test
        fun `should round negative delta mathematically to integer`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("99"),
                    beforePreviousMetricValue = BigDecimal("101"),
                )

            /*
             * (99 - 101) / 101 * 100 = -1.980...
             * После округления получается -2.
             */
            assertThat(result)
                .isEqualByComparingTo("-2")
        }

        @Test
        fun `should compare negative values correctly when more is better`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("-2"),
                    beforePreviousMetricValue = BigDecimal("-3"),
                )

            assertThat(result)
                .isEqualByComparingTo("33")
        }

        @Test
        fun `should compare negative values correctly when less is better`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = LESS_IS_BETTER,
                    submittedMetricValue = BigDecimal("-3"),
                    beforePreviousMetricValue = BigDecimal("-2"),
                )

            assertThat(result)
                .isEqualByComparingTo("50")
        }

        @Test
        fun `should return null when before previous value is absent`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = null,
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should return null when submitted value is zero`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal.ZERO,
                    beforePreviousMetricValue = BigDecimal("100"),
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should return null when before previous value is zero`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = MORE_IS_BETTER,
                    submittedMetricValue = BigDecimal("100"),
                    beforePreviousMetricValue = BigDecimal.ZERO,
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should throw exception when delta direction is unsupported`() {
            assertThatThrownBy {
                calculator.calculateDeltaValue(
                    direction = UNSUPPORTED_DIRECTION,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )
            }
                .isInstanceOf(
                    AiInternalServerException::class.java,
                )
                .hasMessage(
                    "Unsupported metric direction: " +
                        UNSUPPORTED_DIRECTION,
                )
        }

        @Test
        fun `should throw exception when delta direction is null`() {
            assertThatThrownBy {
                calculator.calculateDeltaValue(
                    direction = null,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = BigDecimal("100"),
                )
            }
                .isInstanceOf(
                    AiInternalServerException::class.java,
                )
                .hasMessage(
                    "Unsupported metric direction: null",
                )
        }

        @Test
        fun `should not validate direction when before previous value is null`() {
            val result =
                calculator.calculateDeltaValue(
                    direction = UNSUPPORTED_DIRECTION,
                    submittedMetricValue = BigDecimal("120"),
                    beforePreviousMetricValue = null,
                )

            assertThat(result).isNull()
        }
    }

    @Nested
    inner class CalculateDiagramValueTest {

        @Test
        fun `should return null when actual value is absent`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = null,
                    targetValue = BigDecimal("100"),
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should return null when target value is absent`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("80"),
                    targetValue = null,
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should return null when target value is zero`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("80"),
                    targetValue = BigDecimal.ZERO,
                )

            assertThat(result).isNull()
        }

        @Test
        fun `should return 100 when more is better and actual exceeds target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("120"),
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should return 100 when less is better and actual is below target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("80"),
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should return 100 when more is better and actual equals target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("100"),
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should return 100 when less is better and actual equals target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("100"),
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should return zero when more is better and actual is negative while target is positive`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("-20"),
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should return zero when less is better and actual is positive while target is negative`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("20"),
                    targetValue = BigDecimal("-100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should return zero when more is better and actual is zero with positive target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal.ZERO,
                    targetValue = BigDecimal("100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should return zero when less is better and actual is zero with negative target`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal.ZERO,
                    targetValue = BigDecimal("-100"),
                )

            assertThat(result)
                .isEqualByComparingTo(BigDecimal.ZERO)
        }

        @Test
        fun `should apply 100 percent check before zero check for more is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal.ZERO,
                    targetValue = BigDecimal("-100"),
                )

            /*
             * Сначала проверяется ФАКТ > ПЛАН:
             * 0 > -100, поэтому результат равен 100.
             */
            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should apply 100 percent check before zero check for less is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal.ZERO,
                    targetValue = BigDecimal("100"),
                )

            /*
             * Сначала проверяется ФАКТ < ПЛАН:
             * 0 < 100, поэтому результат равен 100.
             */
            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should calculate diagram value when more is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("40"),
                    targetValue = BigDecimal("80"),
                )

            assertThat(result)
                .isEqualByComparingTo("50")
        }

        @Test
        fun `should calculate diagram value when less is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("100"),
                    targetValue = BigDecimal("40"),
                )

            assertThat(result)
                .isEqualByComparingTo("40")
        }

        @Test
        fun `should round diagram value mathematically when more is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("2"),
                    targetValue = BigDecimal("3"),
                )

            /*
             * 2 / 3 * 100 = 66.66...
             * После HALF_UP округляется до 67.
             */
            assertThat(result)
                .isEqualByComparingTo("67")
        }

        @Test
        fun `should round diagram value mathematically when less is better`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("3"),
                    targetValue = BigDecimal("2"),
                )

            /*
             * 2 / 3 * 100 = 66.66...
             * После HALF_UP округляется до 67.
             */
            assertThat(result)
                .isEqualByComparingTo("67")
        }

        @Test
        fun `should return 100 when more is better and both values are negative but actual is greater`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = MORE_IS_BETTER,
                    actualValue = BigDecimal("-50"),
                    targetValue = BigDecimal("-100"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should return 100 when less is better and both values are negative but actual is less`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = LESS_IS_BETTER,
                    actualValue = BigDecimal("-100"),
                    targetValue = BigDecimal("-50"),
                )

            assertThat(result)
                .isEqualByComparingTo("100")
        }

        @Test
        fun `should throw exception when diagram direction is unsupported`() {
            assertThatThrownBy {
                calculator.calculateDiagramValue(
                    direction = UNSUPPORTED_DIRECTION,
                    actualValue = BigDecimal("80"),
                    targetValue = BigDecimal("100"),
                )
            }
                .isInstanceOf(
                    AiInternalServerException::class.java,
                )
                .hasMessage(
                    "Unsupported metric direction: " +
                        UNSUPPORTED_DIRECTION,
                )
        }

        @Test
        fun `should throw exception when diagram direction is null`() {
            assertThatThrownBy {
                calculator.calculateDiagramValue(
                    direction = null,
                    actualValue = BigDecimal("80"),
                    targetValue = BigDecimal("100"),
                )
            }
                .isInstanceOf(
                    AiInternalServerException::class.java,
                )
                .hasMessage(
                    "Unsupported metric direction: null",
                )
        }

        @Test
        fun `should not validate direction when target value is zero`() {
            val result =
                calculator.calculateDiagramValue(
                    direction = UNSUPPORTED_DIRECTION,
                    actualValue = BigDecimal("80"),
                    targetValue = BigDecimal.ZERO,
                )

            assertThat(result).isNull()
        }
    }

    private companion object {
        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"

        const val UNSUPPORTED_DIRECTION =
            "unsupported_direction"

        const val UNSUPPORTED_DIRECTION_MESSAGE =
            "Unsupported metric direction: {0}"
    }
}

```
