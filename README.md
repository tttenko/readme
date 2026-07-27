```java

@Configuration
class OpenApiGroupsConfig {

    @Bean
    fun cmsAdminOpenApi(): GroupedOpenApi {
        return createRoleGroup(
            groupName = "cms-admin",
            displayName = "Администратор",
            authority = CMS_ADMIN,
        )
    }

    @Bean
    fun projectOfficeOpenApi(): GroupedOpenApi {
        return createRoleGroup(
            groupName = "project-office",
            displayName = "Проектный офис",
            authority = PROJECT_OFFICE,
        )
    }

    private fun createRoleGroup(
        groupName: String,
        displayName: String,
        authority: String,
    ): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group(groupName)
            .displayName(displayName)
            .packagesToScan(CONTROLLERS_PACKAGE)
            .addOpenApiMethodFilter { method ->
                hasAuthority(
                    method = method,
                    authority = authority,
                )
            }
            .build()
    }

    private fun hasAuthority(
        method: Method,
        authority: String,
    ): Boolean {
        val preAuthorizeExpression = findPreAuthorizeExpression(method)
            ?: return false

        return preAuthorizeExpression.contains("'$authority'") ||
            preAuthorizeExpression.contains("\"$authority\"")
    }

    /**
     * Сначала проверяется @PreAuthorize на методе.
     * Если её нет, проверяется аннотация на всём контроллере.
     *
     * Аннотация метода имеет приоритет над аннотацией класса.
     */
    private fun findPreAuthorizeExpression(method: Method): String? {
        val methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(
            method,
            PreAuthorize::class.java,
        )

        if (methodAnnotation != null) {
            return methodAnnotation.value
        }

        return AnnotatedElementUtils.findMergedAnnotation(
            method.declaringClass,
            PreAuthorize::class.java,
        )?.value
    }

    private companion object {
        const val CONTROLLERS_PACKAGE = "ru.sber.prm.controller"

        const val CMS_ADMIN = "CMS_ADMIN"
        const val PROJECT_OFFICE = "PROJECT_OFFICE"
    }
}
```
