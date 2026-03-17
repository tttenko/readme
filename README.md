```java

@Mapping(target = "uuid", ignore = true)
    @Mapping(target = "changedBy", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "changedAt", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    StsDataEntity toEntity(CreateStsDataRequest request);

    StsDataDto toDto(StsDataEntity entity);


```
