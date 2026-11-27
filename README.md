```java
@ParameterizedTest
@ValueSource(strings = ["CMS_ADMIN", "PROJECT_OFFICE", "TRANSFORMATION_OFFICE"])
fun `allowed roles get meta ai objects successfully`(authority: String) {
    every { metaAiObjectsService.getMetaAiObjects(any()) } returns emptyList()

    mockMvc.perform(asUser(authority, get(OBJECTS_URL)))
        .andExpect(status().isOk)
        .andExpect(content().json("[]"))

    verify(exactly = 1) { metaAiObjectsService.getMetaAiObjects(any()) }
}

```
