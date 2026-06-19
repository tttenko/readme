```java
@Test
fun `update should throw exception when planned date is blank`() {
    val agent = agent(id = 1L)

    val request = listOf(
        mapOf(
            "status" to "pilot",
            "plannedDate" to "   ",
        )
    )

    every {
        messageProvider[WRONG_STATUS_SLA_PLANNED_DATE_VALUE]
    } returns "Wrong statusSla plannedDate value"

    val exception = assertThrows<AiBadRequestException> {
        statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )
    }

    assertEquals(
        WRONG_STATUS_SLA_PLANNED_DATE_VALUE,
        exception.errorCode,
    )
}
```
