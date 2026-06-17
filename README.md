```java
@Schema(description = "Запрос на создание enabler")
data class CreateEnablerRequest(

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Название enabler",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Enabler 1",
    )
    val name: String,

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Краткое описание",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Краткое описание enabler",
    )
    val shortDescription: String,

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Описание",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Описание enabler",
    )
    val description: String,

    @field:NotBlank
    @field:Size(max = 2000)
    @field:Schema(
        maxLength = 2000,
        description = "НСсылка на пространство в Confluence",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "https://confluence.example.ru/display/ENABLER",
    )
    val infoLink: String,
)

@Schema(description = "Запрос на обновление enabler")
data class UpdateEnablerRequest(

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Название enabler",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Enabler 1",
    )
    val name: String,

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Краткое описание",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Краткое описание enabler",
    )
    val shortDescription: String,

    @field:NotBlank
    @field:Size(max = 255)
    @field:Schema(
        maxLength = 255,
        description = "Описание",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "Описание enabler",
    )
    val description: String,

    @field:NotBlank
    @field:Size(max = 2000)
    @field:Schema(
        maxLength = 2000,
        description = "НСсылка на пространство в Confluence",
        required = true,
        nullable = false,
        pattern = Metadata.Swagger.ANY_CHAR_PATTERN,
        example = "https://confluence.example.ru/display/ENABLER",
    )
    val infoLink: String,

    @field:Schema(
        description = "Признак отключения enabler",
        required = true,
        nullable = false,
        example = "false",
    )
    val disabled: Boolean,
)


```
