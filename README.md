    ```java
@Repository
interface MetricApplicabilityRequestRepository :
    JpaRepository<MetricApplicabilityRequestEntity, Long>,
    JpaSpecificationExecutor<MetricApplicabilityRequestEntity> {

    /**
     * Получает страницу заявок вместе со связями,
     * необходимыми для формирования ответа.
     *
     * EntityGraph позволяет избежать дополнительных запросов к БД
     * для каждой заявки при обращении к инициативе и метрике.
     */
    @EntityGraph(
        attributePaths = [
            "initiativeMetricAssignment",
            "initiativeMetricAssignment.initiativeMetricType",
            "initiativeMetricAssignment.initiativeMetricType.aiAgent",
            "initiativeMetricAssignment.metric"
        ]
    )
    override fun findAll(
        specification: Specification<MetricApplicabilityRequestEntity>?,
        pageable: Pageable
    ): Page<MetricApplicabilityRequestEntity>

    /**
     * Возвращает количество видимых заявок указанного статуса.
     */
    fun countByStatusAndIsVisibleInOfficeTrue(
        status: MetricApplicabilityRequestStatus
    ): Long

    /**
     * Возвращает общее количество заявок,
     * отображаемых Офису.
     */
    fun countByIsVisibleInOfficeTrue(): Long

    /**
     * Проверяет наличие заявки указанного статуса
     * для конкретного assignment.
     */
    fun existsByInitiativeMetricAssignmentIdAndStatus(
        initiativeMetricAssignmentId: Long,
        status: MetricApplicabilityRequestStatus
    ): Boolean

    /**
     * Получает заявку с блокировкой записи
     * на время транзакции.
     */
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query(
        """
        select r
        from MetricApplicabilityRequestEntity r
        where r.id = :requestId
        """
    )
    fun findByIdForUpdate(
        @Param("requestId")
        requestId: Long
    ): MetricApplicabilityRequestEntity?

    /**
     * Возвращает последнюю заявку assignment.
     *
     * createdAt хранится как DATE, поэтому id используется
     * как дополнительная стабильная сортировка.
     */
    fun findFirstByInitiativeMetricAssignmentIdOrderByCreatedAtDescIdDesc(
        initiativeMetricAssignmentId: Long
    ): MetricApplicabilityRequestEntity?

    /**
     * Возвращает последнюю заявку assignment.
     *
     * Метод из реализации scheduler коллеги.
     */
    @Query(
        """
        select r
        from MetricApplicabilityRequestEntity r
        where r.initiativeMetricAssignment.id = :assignmentId
        order by r.createdAt desc
        """
    )
    fun findLatestByInitiativeMetricAssignmentId(
        @Param("assignmentId")
        assignmentId: Long
    ): MetricApplicabilityRequestEntity?

    /**
     * Возвращает последнюю заявку сразу
     * для каждого assignment из набора.
     *
     * Используется GET /metrics/value.
     *
     * createdAt хранится как DATE, поэтому при одинаковой дате
     * более новая заявка определяется по большему id.
     */
    @Query(
        """
        select request
        from MetricApplicabilityRequestEntity request
        where request.initiativeMetricAssignment.id in :assignmentIds
          and not exists (
              select newerRequest.id
              from MetricApplicabilityRequestEntity newerRequest
              where newerRequest.initiativeMetricAssignment.id =
                    request.initiativeMetricAssignment.id
                and (
                    newerRequest.createdAt > request.createdAt
                    or (
                        newerRequest.createdAt = request.createdAt
                        and newerRequest.id > request.id
                    )
                )
          )
        """
    )
    fun findLatestByAssignmentIds(
        @Param("assignmentIds")
        assignmentIds: Set<Long>
    ): List<MetricApplicabilityRequestEntity>

    /**
     * Переводит assignment в ACTIVE
     * при наступлении resumePeriod.
     */
    @Modifying
    @Transactional
    @Query(
        value = """
            update initiative_metric_assignment ima
            set applicability_status = 'ACTIVE'
            where ima.id in (
                select mr.initiative_metric_assignment_id
                from metric_applicability_request mr
                inner join initiative_metric_assignment ima2
                    on ima2.id = mr.initiative_metric_assignment_id
                where ima2.applicability_status in ('PENDING', 'NOT_APPLICABLE')
                  and mr.resume_period is not null
                  and extract(year from mr.resume_period) = extract(year from current_date)
                  and extract(month from mr.resume_period) = extract(month from current_date)
                  and mr.id = (
                      select r2.id
                      from metric_applicability_request r2
                      where r2.initiative_metric_assignment_id = ima2.id
                      order by r2.created_at desc
                      limit 1
                  )
            )
        """,
        nativeQuery = true
    )
    fun updateAssignmentsToActive(): Int

    /**
     * Обновляет заявку при наступлении resumePeriod.
     */
    @Modifying
    @Transactional
    @Query(
        value = """
            update metric_applicability_request mar
            set effective_to_period = mar.resume_period,
                is_visible_in_office = false
            where mar.id in (
                select mr.id
                from metric_applicability_request mr
                inner join initiative_metric_assignment ima
                    on ima.id = mr.initiative_metric_assignment_id
                where ima.applicability_status in ('PENDING', 'NOT_APPLICABLE')
                  and mr.resume_period is not null
                  and extract(year from mr.resume_period) = extract(year from current_date)
                  and extract(month from mr.resume_period) = extract(month from current_date)
                  and mr.id = (
                      select r2.id
                      from metric_applicability_request r2
                      where r2.initiative_metric_assignment_id = ima.id
                      order by r2.created_at desc
                      limit 1
                  )
            )
        """,
        nativeQuery = true
    )
    fun updateRequests(): Int

    /**
     * Возвращает заявки,
     * обработанные scheduler по resumePeriod.
     */
    @Transactional(readOnly = true)
    @Query(
        """
        select distinct r
        from MetricApplicabilityRequestEntity r
        join fetch r.initiativeMetricAssignment ima
        join fetch ima.initiativeMetricType imt
        join fetch imt.aiAgent
        join fetch ima.metric
        where r.effectiveToPeriod = r.resumePeriod
          and r.isVisibleInOffice = false
          and r.resumePeriod is not null
          and extract(year from r.resumePeriod) = extract(year from current_date)
          and extract(month from r.resumePeriod) = extract(month from current_date)
        """
    )
    fun findUpdatedRequests(): List<MetricApplicabilityRequestEntity>
}
```
