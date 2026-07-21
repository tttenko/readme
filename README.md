```java


@ExtendWith(MockKExtension::class)
class MetricsAdminServiceTest {

    @MockK
    private lateinit var metricsDirectoryRepository:
        MetricsDirectoryRepository

    @MockK
    private lateinit var messageProvider:
        MessageProvider

    @MockK(relaxed = true)
    private lateinit var userInfoProvider:
        UserInfoProvider

    @MockK(relaxed = true)
    private lateinit var dateTimeProvider:
        DateTimeProvider

    private lateinit var metric:
        MetricsDirectoryEntity

    private lateinit var service:
        MetricsAdminService

    private var storedCode: String? = null

    private var storedPreAnalytics: Boolean? = null

    @BeforeEach
    fun setUp() {
        storedCode = null
        storedPreAnalytics = null

        metric =
            mockk(relaxed = true) {
                every {
                    id
                } returns METRIC_ID

                every {
                    code
                } answers {
                    storedCode
                }

                every {
                    code = any()
                } answers {
                    storedCode = firstArg()
                }

                every {
                    isPreAnalytics
                } answers {
                    storedPreAnalytics
                }

                every {
                    isPreAnalytics = any()
                } answers {
                    storedPreAnalytics = firstArg()
                }
            }

        service =
            MetricsAdminService(
                metricsDirectoryRepository =
                    metricsDirectoryRepository,
                messageProvider = messageProvider,
                userInfoProvider = userInfoProvider,
                dateTimeProvider = dateTimeProvider,
            )

        every {
            metricsDirectoryRepository.findById(
                METRIC_ID,
            )
        } returns Optional.of(metric)

        every {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = any(),
                    id = METRIC_ID,
                )
        } returns false

        every {
            metricsDirectoryRepository.save(metric)
        } returns metric

        every {
            messageProvider[METRIC_NOT_FOUND]
        } returns
            "Метрика с идентификатором {0} не найдена"

        every {
            messageProvider[
                PRE_ANALYTICS_CODE_REQUIRED
            ]
        } returns
            "Для метрики pre-analytics необходимо заполнить code"

        every {
            messageProvider[
                METRIC_CODE_ALREADY_EXISTS
            ]
        } returns
            "Метрика с code {0} уже существует"
    }

    @Test
    fun `should throw not found when metric does not exist`() {
        every {
            metricsDirectoryRepository.findById(
                METRIC_ID,
            )
        } returns Optional.empty()

        assertThatThrownBy {
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = METRIC_CODE,
                        isPreAnalytics = true,
                    ),
            )
        }
            .isInstanceOf(
                AiNotFoundException::class.java,
            )
            .hasMessage(
                "Метрика с идентификатором " +
                    "$METRIC_ID не найдена",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())

            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    any(),
                    any(),
                )

            userInfoProvider.currentUser()

            dateTimeProvider.currentDateTime()
        }
    }

    @Test
    fun `should throw bad request when pre analytics enabled without code`() {
        assertThatThrownBy {
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = null,
                        isPreAnalytics = true,
                    ),
            )
        }
            .isInstanceOf(
                AiBadRequestException::class.java,
            )
            .hasMessage(
                "Для метрики pre-analytics " +
                    "необходимо заполнить code",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())

            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    any(),
                    any(),
                )

            userInfoProvider.currentUser()

            dateTimeProvider.currentDateTime()
        }
    }

    @Test
    fun `should throw bad request when pre analytics enabled with blank code`() {
        assertThatThrownBy {
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = "   ",
                        isPreAnalytics = true,
                    ),
            )
        }
            .isInstanceOf(
                AiBadRequestException::class.java,
            )
            .hasMessage(
                "Для метрики pre-analytics " +
                    "необходимо заполнить code",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())

            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    any(),
                    any(),
                )

            userInfoProvider.currentUser()

            dateTimeProvider.currentDateTime()
        }
    }

    @Test
    fun `should throw bad request when metric code already exists`() {
        every {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = METRIC_CODE,
                    id = METRIC_ID,
                )
        } returns true

        assertThatThrownBy {
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = METRIC_CODE,
                        isPreAnalytics = true,
                    ),
            )
        }
            .isInstanceOf(
                AiBadRequestException::class.java,
            )
            .hasMessage(
                "Метрика с code $METRIC_CODE " +
                    "уже существует",
            )

        verify(exactly = 1) {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = METRIC_CODE,
                    id = METRIC_ID,
                )
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())

            userInfoProvider.currentUser()

            dateTimeProvider.currentDateTime()
        }
    }

    @Test
    fun `should update pre analytics settings`() {
        val request =
            request(
                code = METRIC_CODE,
                isPreAnalytics = true,
            )

        val response =
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request = request,
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                UpdateMetricPreAnalyticsResponse(
                    metricId = METRIC_ID,
                    code = METRIC_CODE,
                    isPreAnalytics = true,
                ),
            )

        verify(exactly = 1) {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = METRIC_CODE,
                    id = METRIC_ID,
                )

            userInfoProvider.currentUser()

            dateTimeProvider.currentDateTime()

            metricsDirectoryRepository.save(metric)
        }

        verify(exactly = 1) {
            metric.code = METRIC_CODE
            metric.isPreAnalytics = true
            metric.updatedBy = any()
            metric.updatedAt = any()
        }
    }

    @Test
    fun `should save null settings`() {
        storedCode = METRIC_CODE
        storedPreAnalytics = true

        val response =
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = null,
                        isPreAnalytics = null,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                UpdateMetricPreAnalyticsResponse(
                    metricId = METRIC_ID,
                    code = null,
                    isPreAnalytics = null,
                ),
            )

        verify(exactly = 0) {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    any(),
                    any(),
                )
        }

        verify(exactly = 1) {
            metric.code = null
            metric.isPreAnalytics = null
            metricsDirectoryRepository.save(metric)
        }
    }

    @Test
    fun `should allow disabled pre analytics without code`() {
        val response =
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = null,
                        isPreAnalytics = false,
                    ),
            )

        assertThat(response.metricId)
            .isEqualTo(METRIC_ID)

        assertThat(response.code)
            .isNull()

        assertThat(response.isPreAnalytics)
            .isFalse()

        verify(exactly = 0) {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    any(),
                    any(),
                )
        }

        verify(exactly = 1) {
            metric.code = null
            metric.isPreAnalytics = false
            metricsDirectoryRepository.save(metric)
        }
    }

    @Test
    fun `should check code uniqueness when pre analytics flag is null`() {
        val response =
            service.updatePreAnalyticsSettings(
                metricId = METRIC_ID,
                request =
                    request(
                        code = METRIC_CODE,
                        isPreAnalytics = null,
                    ),
            )

        assertThat(response.code)
            .isEqualTo(METRIC_CODE)

        assertThat(response.isPreAnalytics)
            .isNull()

        verify(exactly = 1) {
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = METRIC_CODE,
                    id = METRIC_ID,
                )

            metricsDirectoryRepository.save(metric)
        }
    }

    private fun request(
        code: String?,
        isPreAnalytics: Boolean?,
    ): UpdateMetricPreAnalyticsRequest {
        return UpdateMetricPreAnalyticsRequest(
            code = code,
            isPreAnalytics = isPreAnalytics,
        )
    }

    private companion object {

        val METRIC_ID: UUID =
            UUID.fromString(
                "a860b390-b739-48e6-a694-e96582eb4e95",
            )

        const val METRIC_CODE =
            "accuracy"

        const val PRE_ANALYTICS_CODE_REQUIRED =
            "metric.pre-analytics.code-required"

        const val METRIC_CODE_ALREADY_EXISTS =
            "metric.code.already-exists"
    }
}

```
