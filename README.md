```java

@Test
fun `should skip initiative when block and division cannot be resolved`() {
    val fields = mockk<SearchIssueFieldsDto> {
        every { customfield_30000 } returns emptyList()
        every { customfield_30001 } returns emptyList()
    }

    val issue = mockk<SearchIssueDto> {
        every { key } returns "CROSSGOAL-300"
        every { this@mockk.fields } returns fields
    }

    val referenceData = JiraImportReferenceData(
        strategiesByJiraKey = emptyMap(),
        enablersByNormalizedName = emptyMap(),
        statusesByCode = emptyMap(),
        qualityGates = emptyList(),
        divisionsByLabel = emptyMap(),
        blocksByLabel = emptyMap(),
        initiativeTypesByCode = emptyMap(),
    )

    every {
        organizationResolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = emptyList(),
            referenceData = referenceData,
        )
    } returns JiraInitiativeOrganization(
        division = null,
        block = null,
    )

    val result =
        service.createInitiativeFromJira(
            issue = issue,
            referenceData = referenceData,
        )

    assertNull(result)

    verify(exactly = 0) {
        agentRepository.save(any())
    }
}

```
