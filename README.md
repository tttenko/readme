```java
@Test
void givenSupplierUserType_whenToApproveStsData_thenAllowed() {
    // given
    AuthorizedUser principal = mock(AuthorizedUser.class);
    when(principal.getUserType())
            .thenReturn(ru.sber.cs.supplier.portal.authorization.dto.enums.UserType.SUPPLIER);

    UUID uuid = UUID.randomUUID();

    ToApproveStsDataRequest request = new ToApproveStsDataRequest();
    request.setUuids(List.of(uuid));

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.TO_APPROVE_IN);

    StsDataDto dto = new StsDataDto();
    dto.setUuid(uuid);
    dto.setStatusId(StsStatus.TO_APPROVE_IN);

    StsBatchOperationResult<StsDataEntity> serviceResult =
            new StsBatchOperationResult<>(List.of(entity), List.of());

    when(stsWorkFlowService.toApprove(List.of(uuid))).thenReturn(serviceResult);
    when(stsDataMapper.toDto(entity)).thenReturn(dto);

    // when
    ResultObj<ToApproveStsDataResultDto> result =
            stsDataController.toApproveStsData(principal, request);

    // then
    assertThat(result).isNotNull();
    assertThat(result.getData()).isNotNull();
    assertThat(result.getData().getProcessed()).containsExactly(dto);
    assertThat(result.getData().getErrors()).isEmpty();

    verify(stsWorkFlowService).toApprove(List.of(uuid));
    verify(stsDataMapper).toDto(entity);
    verifyNoMoreInteractions(stsWorkFlowService, stsDataMapper);
}
```
