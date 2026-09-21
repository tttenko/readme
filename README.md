```java
fun saveInitiativeMetricValue(
        initiativeId: Long,
        request: SaveInitiativeMetricValuesRequest,
    ): ResponseEntity<SaveInitiativeMetricValueResponse> {

        val response =
            initiativeMetricValueCreator
                .saveInitiativeMetricValue(
                    initiativeId = initiativeId,
                    request = request,
                )

        return if (
            response.code ==
            SaveInitiativeMetricValueResponse.METRIC_UNLINK_CODE
        ) {
            ResponseEntity
                .badRequest()
                .body(response)
        } else {
            ResponseEntity.ok(response)
        }
    }




```
