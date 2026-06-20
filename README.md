```java
@DeleteMapping
@ResponseStatus(HttpStatus.NO_CONTENT)
@Operation(
    summary = "Удаление метрики",
    responses = [
        ApiResponse(responseCode = "204"),
        ApiResponse(responseCode = "409"),
    ],
)
@ExceptionApiResponses
@PreAuthorize(value = "hasAuthority('TRANSFORMATION_OFFICE')")
fun deleteMetric(
    @RequestParam metricId: UUID,
) {
    metricsService.deleteMetric(
        metricId = metricId,
    )
}
```
