```java

private fun cmsAdminUser() = userWithRole("CMS_ADMIN")

private fun projectOfficeUser() = userWithRole("PROJECT_OFFICE")

private fun transformationOfficeUser() =
    userWithRole("TRANSFORMATION_OFFICE")

private fun userWithRole(role: String) = UserDto(
    id = 10L,
    roles = setOf(role),
    email = null,
    login = null,
    firstName = null,
    lastName = null,
    patronymic = null,
    phoneNumber = null,
    position = null,
    sberbankEmployee = null,
    companyId = null,
)

```
