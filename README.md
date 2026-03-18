```java

@GetMapping("/contract/{contractUuid}/sts")
    @Operation(
            operationId = "getStsDataByContractUuid",
            summary = "Получение страницы СТС по contractUuid"
    )
    @ApiResponse(responseCode = "200", description = "Страница СТС успешно получена")
    @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса")
    @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    ResultObj<PageDto<StsDataDto>> getStsDataByContractUuid(
            @PathVariable("contractUuid")
            @NotNull(message = "Параметр contractUuid не должен быть null")
            UUID contractUuid,

            @RequestParam(name = "$filter", required = false)
            String filter,

            @RequestParam(name = "$orderby", required = false)
            String orderBy,

            @RequestParam(name = "$top")
            @Min(value = 1, message = "Параметр $top должен быть не меньше 1")
            @Max(value = 200, message = "Параметр $top должен быть не больше 200")
            Integer top,

            @RequestParam(name = "$skip")
            @Min(value = 0, message = "Параметр $skip должен быть не меньше 0")
            Integer skip
    );


     @Override
    public ResultObj<PageDto<StsDataDto>> getStsDataByContractUuid(UUID contractUuid,
                                                                    String filter,
                                                                    String orderBy,
                                                                    Integer top,
                                                                    Integer skip) {
        return getSuccessPageResponse(
                stsDataService.getPageByContractUuid(contractUuid, filter, orderBy, top, skip)
        );
    }

    @Transactional(readOnly = true)
    public PageDto<StsDataDto> getPageByContractUuid(UUID contractUuid,
                                                     String filter,
                                                     String orderBy,
                                                     Integer top,
                                                     Integer skip) {

        Pageable pageable = PageRequest.of(
                skip / top,
                top,
                Sort.by(Sort.Direction.DESC, "createdAt")
        );

        Page<StsDataEntity> page = stsDataRepository.findAllByContractUuidAndDeleted(contractUuid, false, pageable);

        PageDto<StsDataDto> result = new PageDto<>();
        result.setItems(page.getContent().stream().map(stsDataMapper::toDto).toList());
        result.setPage(page.getNumber());
        result.setSize(page.getSize());
        result.setTotalElements(page.getTotalElements());
        result.setTotalPages(page.getTotalPages());
        result.setFirst(page.isFirst());
        result.setLast(page.isLast());

        return result;
    }

    @Transactional(readOnly = true)
    public PageDto<StsDataDto> getPageByContractUuid(UUID contractUuid,
                                                     String filter,
                                                     String orderBy,
                                                     Integer top,
                                                     Integer skip) {

        Pageable pageable = PageRequest.of(
                skip / top,
                top,
                Sort.by(Sort.Direction.DESC, "createdAt")
        );

        Page<StsDataEntity> page = stsDataRepository.findAllByContractUuidAndDeleted(contractUuid, false, pageable);

        PageDto<StsDataDto> result = new PageDto<>();
        result.setItems(page.getContent().stream().map(stsDataMapper::toDto).toList());
        result.setPage(page.getNumber());
        result.setSize(page.getSize());
        result.setTotalElements(page.getTotalElements());
        result.setTotalPages(page.getTotalPages());
        result.setFirst(page.isFirst());
        result.setLast(page.isLast());

        return result;
    }

    public static <T> ResultObj<PageDto<T>> getSuccessPageResponse(PageDto<T> page) {
        ResultObj<PageDto<T>> result = new ResultObj<>();
        result.addMessages(success("Поиск выполнен", null, "Найдено записей: " + page.getTotalElements()));
        result.setData(page);
        result.setCount(page.getTotalElements());
        return result;
    }
```
