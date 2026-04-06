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

    when(stsDataService.toApprove(uuids)).thenReturn(List.of(firstEntity, secondEntity));
    when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
    when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

    // when
    ResultObj<List<StsDataDto>> actualResult = stsDataController.toApproveStsData(principal, request);

    // then
    assertThat(actualResult).isNotNull();
    assertThat(actualResult.getCount()).isEqualTo(2L);
    assertThat(actualResult.getData()).hasSize(2);
    assertThat(actualResult.getData().get(0)).isSameAs(firstDto);
    assertThat(actualResult.getData().get(1)).isSameAs(secondDto);

    verify(stsDataService).toApprove(uuids);
    verify(stsDataMapper).toDto(firstEntity);
    verify(stsDataMapper).toDto(secondEntity);
    verifyNoMoreInteractions(stsDataService, stsDataMapper);
}

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = StsDataControllerImplMethodSecurityTest.TestConfig.class)
class StsDataControllerImplMethodSecurityTest {

    @jakarta.annotation.Resource
    private StsDataController stsDataController;

    @jakarta.annotation.Resource
    private StsDataService stsDataService;

    @jakarta.annotation.Resource
    private StsDataMapper stsDataMapper;

    @BeforeEach
    void setUp() {
        reset(stsDataService, stsDataMapper);

        SecurityContextHolder.clearContext();
        SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken("test-user", "test-password", List.of())
        );
    }

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

        when(stsDataService.toApprove(List.of(uuid))).thenReturn(List.of(entity));
        when(stsDataMapper.toDto(entity)).thenReturn(dto);

        // when
        ResultObj<List<StsDataDto>> result = stsDataController.toApproveStsData(principal, request);

        // then
        assertThat(result).isNotNull();
        assertThat(result.getCount()).isEqualTo(1L);
        assertThat(result.getData()).containsExactly(dto);

        verify(stsDataService).toApprove(List.of(uuid));
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenInternalUserType_whenToApproveStsData_thenAccessDenied() {
        // given
        AuthorizedUser principal = mock(AuthorizedUser.class);
        when(principal.getUserType())
                .thenReturn(ru.sber.cs.supplier.portal.authorization.dto.enums.UserType.INTERNAL);

        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(UUID.randomUUID()));

        // when / then
        assertThrows(
                AccessDeniedException.class,
                () -> stsDataController.toApproveStsData(principal, request)
        );

        verifyNoInteractions(stsDataService, stsDataMapper);
    }

    @Configuration
    @EnableMethodSecurity
    static class TestConfig {

        @Bean
        StsDataService stsDataService() {
            return mock(StsDataService.class);
        }

        @Bean
        StsDataMapper stsDataMapper() {
            return mock(StsDataMapper.class);
        }

        @Bean
        StsDataController stsDataController(StsDataService stsDataService,
                                            StsDataMapper stsDataMapper) {
            return new StsDataControllerImpl(stsDataService, stsDataMapper);
        }
    }
}
```
