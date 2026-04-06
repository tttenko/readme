```java
@Test
void givenValidRequest_whenToApproveStsData_thenReturnOk() throws Exception {
    // given
    UUID firstUuid = UUID.randomUUID();
    UUID secondUuid = UUID.randomUUID();
    UUID contractUuid = UUID.randomUUID();
    UUID userUuid = UUID.fromString("00000000-0000-0000-0000-000000000001");

    ToApproveStsDataRequest request = new ToApproveStsDataRequest();
    request.setUuids(List.of(firstUuid, secondUuid));

    StsDataEntity firstEntity = new StsDataEntity();
    firstEntity.setUuid(firstUuid);

    StsDataEntity secondEntity = new StsDataEntity();
    secondEntity.setUuid(secondUuid);

    StsDataDto firstDto = new StsDataDto();
    firstDto.setUuid(firstUuid);
    firstDto.setContractUuid(contractUuid);
    firstDto.setTbCode("1234");
    firstDto.setVehicleNumber("A123AA777");
    firstDto.setVehicleBrand("КамАЗ");
    firstDto.setComment("Первая запись");
    firstDto.setStatusId(StsStatus.TO_APPROVE_IN);
    firstDto.setCreatedBy(userUuid);
    firstDto.setUpdatedBy(userUuid);
    firstDto.setDeleted(false);

    StsDataDto secondDto = new StsDataDto();
    secondDto.setUuid(secondUuid);
    secondDto.setContractUuid(contractUuid);
    secondDto.setTbCode("4321");
    secondDto.setVehicleNumber("B777BB777");
    secondDto.setVehicleBrand("МАЗ");
    secondDto.setComment("Вторая запись");
    secondDto.setStatusId(StsStatus.TO_APPROVE_OUT);
    secondDto.setCreatedBy(userUuid);
    secondDto.setUpdatedBy(userUuid);
    secondDto.setDeleted(false);

    StsBatchOperationResult<StsDataEntity> serviceResult =
            new StsBatchOperationResult<>(List.of(firstEntity, secondEntity), List.of());

    when(stsWorkFlowService.toApprove(List.of(firstUuid, secondUuid)))
            .thenReturn(serviceResult);
    when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
    when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

    // when / then
    mockMvc.perform(patch("/ui/v1/sts/to_approve")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.data.processed.length()").value(2))
            .andExpect(jsonPath("$.data.errors.length()").value(0))
            .andExpect(jsonPath("$.data.processed[0].uuid").value(firstUuid.toString()))
            .andExpect(jsonPath("$.data.processed[0].statusId").value("TO_APPROVE_IN"))
            .andExpect(jsonPath("$.data.processed[1].uuid").value(secondUuid.toString()))
            .andExpect(jsonPath("$.data.processed[1].statusId").value("TO_APPROVE_OUT"));

    verify(stsWorkFlowService).toApprove(List.of(firstUuid, secondUuid));
    verify(stsDataMapper).toDto(firstEntity);
    verify(stsDataMapper).toDto(secondEntity);
    verifyNoMoreInteractions(stsWorkFlowService, stsDataMapper);
}
```
