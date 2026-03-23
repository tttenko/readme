```java
@SpringBootTest(classes = OpenApiConfigTest.TestApp.class)
class OpenApiConfigTest {

    @Autowired
    private OpenApiConfig openApiConfig;

    @Test
    void shouldLoadOpenApiConfig() {
        assertThat(openApiConfig).isNotNull();
    }

    @Test
    void shouldHaveOpenApiDefinitionAnnotation() {
        OpenAPIDefinition annotation = OpenApiConfig.class.getAnnotation(OpenAPIDefinition.class);

        assertThat(annotation).isNotNull();

        Info info = annotation.info();
        assertThat(info).isNotNull();
        assertThat(info.title()).isEqualTo("Управление спец. автотранспортом");
        assertThat(info.description()).contains("Сервис предназначен для управления данными по спецавтотранспорту");
        assertThat(info.description()).contains("получения и изменения списка СТС по договору");
        assertThat(info.description()).contains("добавления, редактирования и удаления записей СТС");
        assertThat(info.description()).contains("отправки изменений на согласование");
        assertThat(info.description()).contains("ручного согласования или отклонения изменений сотрудником банка");
        assertThat(info.description()).contains("просмотра истории изменений");
        assertThat(info.description()).contains("интеграции с внешними сервисами статусов и истории изменений");
        assertThat(info.version()).isEqualTo("${spring.application.version}");
    }

    @Test
    void shouldHaveSecuritySchemeAnnotation() {
        SecurityScheme annotation = OpenApiConfig.class.getAnnotation(SecurityScheme.class);

        assertThat(annotation).isNotNull();
        assertThat(annotation.name()).isEqualTo(Headers.AUTH_HEADER);
        assertThat(annotation.type().name()).isEqualTo("APIKEY");
        assertThat(annotation.in().name()).isEqualTo("HEADER");
    }

    @SpringBootConfiguration
    @EnableAutoConfiguration(exclude = {
            DataSourceAutoConfiguration.class,
            HibernateJpaAutoConfiguration.class
    })
    @Import(OpenApiConfig.class)
    static class TestApp {
    }
}
```
