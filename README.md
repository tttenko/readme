```java

@Import(value = [EnablerController::class])
@ContextConfiguration(classes = [EnablerControllerTest.Config::class])
internal class EnablerControllerTest : ControllerTestBase() {

    @Autowired
    private lateinit var service: EnablerService

    @Autowired
    private lateinit var mapper: ObjectMapper

    @TestConfiguration
    internal class Config {

        @Bean
        open fun service() = mockk<EnablerService>()
    }

    @BeforeEach
    fun setUp() {
        clearMocks(service)
    }

    @Test
    @WithMockUser(authorities = ["CMS_ADMIN"])
    fun `should get enabler references with success by default`() {
        // given
        val expectedResult = listOf(
            EnablerResponse(
                id = 1L,
                name = "Enabler 1",
                disabled = false
            ),
            EnablerResponse(
                id = 2L,
                name = "Enabler 2",
                disabled = false
            )
        )

        every { service.getEnablerReferences(false) } returns expectedResult

        // when / then
        mockMvc.perform(MockMvcRequestBuilders.get(API_URL))
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.jsonPath("$[0].id", equalTo(1)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[0].name", equalTo("Enabler 1")))
            .andExpect(MockMvcResultMatchers.jsonPath("$[0].disabled", equalTo(false)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[1].id", equalTo(2)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[1].name", equalTo("Enabler 2")))
            .andExpect(MockMvcResultMatchers.jsonPath("$[1].disabled", equalTo(false)))

        verify(exactly = 1) {
            service.getEnablerReferences(false)
        }
    }

    @Test
    @WithMockUser(authorities = ["PROJECT_OFFICE"])
    fun `should get enabler references with disabled with success`() {
        // given
        val expectedResult = listOf(
            EnablerResponse(
                id = 1L,
                name = "Enabler 1",
                disabled = false
            ),
            EnablerResponse(
                id = 2L,
                name = "Enabler 2",
                disabled = true
            )
        )

        every { service.getEnablerReferences(true) } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.get(API_URL)
                .param("includeDisabled", "true")
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.jsonPath("$[0].id", equalTo(1)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[0].disabled", equalTo(false)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[1].id", equalTo(2)))
            .andExpect(MockMvcResultMatchers.jsonPath("$[1].disabled", equalTo(true)))

        verify(exactly = 1) {
            service.getEnablerReferences(true)
        }
    }

    @Test
    @WithMockUser(authorities = ["CMS_ADMIN"])
    fun `should get empty enabler references with success`() {
        // given
        every { service.getEnablerReferences(false) } returns emptyList()

        // when / then
        mockMvc.perform(MockMvcRequestBuilders.get(API_URL))
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.content().json("[]"))

        verify(exactly = 1) {
            service.getEnablerReferences(false)
        }
    }

    @Test
    @WithMockUser(authorities = ["CMS_ADMIN"])
    fun `should create enabler with success`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        val expectedResult = CreateEnablerResponse(
            id = 1L
        )

        every { service.createEnabler(request) } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL)
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(request))
        )
            .andExpect(MockMvcResultMatchers.status().isCreated)
            .andExpect(MockMvcResultMatchers.jsonPath("$.id", equalTo(1)))

        verify(exactly = 1) {
            service.createEnabler(request)
        }
    }

    @Test
    @WithMockUser(authorities = ["CMS_ADMIN"])
    fun `should update enabler with success`() {
        // given
        val givenId = 1L

        val request = UpdateEnablerRequest(
            name = "Updated enabler",
            disabled = true
        )

        every {
            service.updateEnabler(
                id = givenId,
                request = request
            )
        } just runs

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.put("$API_URL/$givenId")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(request))
        )
            .andExpect(MockMvcResultMatchers.status().isOk)

        verify(exactly = 1) {
            service.updateEnabler(
                id = givenId,
                request = request
            )
        }
    }

    @Test
    @WithMockUser(authorities = ["PROJECT_OFFICE"])
    fun `should return forbidden when project office tries to create enabler`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL)
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(request))
        )
            .andExpect(MockMvcResultMatchers.status().isForbidden)

        verify(exactly = 0) {
            service.createEnabler(any())
        }
    }

    @Test
    @WithMockUser(authorities = ["PROJECT_OFFICE"])
    fun `should return forbidden when project office tries to update enabler`() {
        // given
        val request = UpdateEnablerRequest(
            name = "Updated enabler",
            disabled = false
        )

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.put("$API_URL/1")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(request))
        )
            .andExpect(MockMvcResultMatchers.status().isForbidden)

        verify(exactly = 0) {
            service.updateEnabler(any(), any())
        }
    }

    private companion object {
        private const val API_URL = "/api/ai/v1/reference/enabler"
    }
}
```
