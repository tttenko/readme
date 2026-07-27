```java

@Configuration
class OpenApiGroupsConfig {

    @Bean
    fun adminAndTransformationOfficeOpenApi(): GroupedOpenApi {
        return createRoleGroup(
            groupName = "admin-transformation-office",
            displayName = "Администратор и офис трансформации",
            authorities = setOf(
                CMS_ADMIN,
                TRANSFORMATION_OFFICE,
            ),
        )
    }

    @Bean
    fun projectOfficeOpenApi(): GroupedOpenApi {
        return createRoleGroup(
            groupName = "project-office",
            displayName = "Проектный офис",
            authorities = setOf(PROJECT_OFFICE),
        )
    }

    private fun createRoleGroup(
        groupName: String,
        displayName: String,
        authorities: Set<String>,
    ): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group(groupName)
            .displayName(displayName)
            .packagesToScan(CONTROLLERS_PACKAGE)
            .addOpenApiMethodFilter { method ->
                hasAnyAuthority(
                    method = method,
                    authorities = authorities,
                )
            }
            .build()
    }

    private fun hasAnyAuthority(
        method: Method,
        authorities: Set<String>,
    ): Boolean {
        val preAuthorizeExpression = findPreAuthorizeExpression(method)
            ?: return false

        return authorities.any { authority ->
            containsAuthority(
                expression = preAuthorizeExpression,
                authority = authority,
            )
        }
    }

    private fun containsAuthority(
        expression: String,
        authority: String,
    ): Boolean {
        return expression.contains("'$authority'") ||
            expression.contains("\"$authority\"")
    }

    /**
     * Сначала ищется @PreAuthorize на методе.
     * Если на методе аннотации нет, проверяется контроллер.
     */
    private fun findPreAuthorizeExpression(method: Method): String? {
        val methodAnnotation =
            AnnotatedElementUtils.findMergedAnnotation(
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
        const val TRANSFORMATION_OFFICE = "TRANSFORMATION_OFFICE"
        const val PROJECT_OFFICE = "PROJECT_OFFICE"
    }
}

```
