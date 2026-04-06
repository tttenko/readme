```java
@Test
void givenPrincipalAndValidRequest_whenToApproveStsData_thenReturnWrappedMappedList() {
    // given
    AuthorizedUser principal = mock(AuthorizedUser.class);

    UUID firstUuid = UUID.randomUUID();
    UUID secondUuid = UUID.randomUUID();
    List<UUID> uuids = List.of(firstUuid, secondUuid);

    ToApproveStsDataRequest request = new ToApproveStsDataRequest();
    request.setUuids(uuids);

    StsDataEntity firstEntity = new StsDataEntity();
    firstEntity.setUuid(firstUuid);
    firstEntity.setStatusId(StsStatus.TO_APPROVE_IN);

    StsDataEntity secondEntity = new StsDataEntity();
    secondEntity.setUuid(secondUuid);
    secondEntity.setStatusId(StsStatus.TO_APPROVE_OUT);

    StsDataDto firstDto = new StsDataDto();
    firstDto.setUuid(firstUuid);
    firstDto.setStatusId(StsStatus.TO_APPROVE_IN);

    StsDataDto secondDto = new StsDataDto();
    secondDto.setUuid(secondUuid);
    secondDto.setStatusId(StsStatus.TO_APPROVE_OUT);

    StsBatchOperationResult<StsDataEntity> serviceResult =
            new StsBatchOperationResult<>(List.of(firstEntity, secondEntity), List.of());

    when(stsWorkFlowService.toApprove(uuids)).thenReturn(serviceResult);
    when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
    when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

    // when
    ResultObj<ToApproveStsDataResultDto> actualResult =
            stsDataController.toApproveStsData(principal, request);

    // then
    assertThat(actualResult).isNotNull();
    assertThat(actualResult.getData()).isNotNull();
    assertThat(actualResult.getData().getProcessed()).hasSize(2);
    assertThat(actualResult.getData().getProcessed().get(0)).isSameAs(firstDto);
    assertThat(actualResult.getData().getProcessed().get(1)).isSameAs(secondDto);
    assertThat(actualResult.getData().getErrors()).isEmpty();

    verify(stsWorkFlowService).toApprove(uuids);
    verify(stsDataMapper).toDto(firstEntity);
    verify(stsDataMapper).toDto(secondEntity);
    verifyNoMoreInteractions(stsWorkFlowService, stsDataMapper);
}
```
