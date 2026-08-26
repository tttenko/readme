```java
@Suppress("UNCHECKED_CAST")
@SpringBootTest(
    properties = [
        "spring.liquibase.enabled=true",
    ],
)
@AutoConfigureEmbeddedDatabase(
    provider = AutoConfigureEmbeddedDatabase.DatabaseProvider.ZONKY,
    type = AutoConfigureEmbeddedDatabase.DatabaseType.POSTGRES,
)
@EnableWebSecurity
@ContextConfiguration(
    classes = [IntegrationTestConfig::class],
)
@EnableAutoConfiguration(
    exclude = [
        SecurityAutoConfiguration::class,
        ManagementWebSecurityAutoConfiguration::class,
    ],
)
@ExtendWith(SpringExtension::class)
abstract class AbstractJUnitIntegrationTest {

    @Autowired
    lateinit var mapper: ObjectMapper

    @Autowired
    lateinit var mockMvc: MockMvc

    @Autowired
    lateinit var sqlExecutor: SqlExecutor

    @MockkBean(relaxed = true)
    lateinit var userInfoProvider: UserInfoProvider

    @MockkBean
    lateinit var dateTimeProvider: DateTimeProvider

    fun executeSqlScript(
        filePath: String,
    ) {
        sqlExecutor.executeSqlScript(filePath)
    }

    fun clearTables() {
        sqlExecutor.executeSqlScript(
            "/integration/clearTables.sql"
        )
    }
}

```
