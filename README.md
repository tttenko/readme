```java
private StsBatchOperationErrorDto toErrorDto(StsBatchOperationError error) {
    StsBatchOperationErrorDto dto = new StsBatchOperationErrorDto();
    dto.setUuid(error.uuid());
    dto.setCode(error.code());
    dto.setMessage(error.message());
    return dto;

```
