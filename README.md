```java

@Modifying
@Query(
    nativeQuery = true,
    value = """
        insert into initiative_metric_value (
            initiative_agent_type_id,
            metric_directory_id,
            period_month,
            metric_value,
            target_value
        )
        select
            previous.initiative_agent_type_id,
            previous.metric_directory_id,
            :currentPeriod,
            case
                when directory.frequency = 'constant'
                    then previous.metric_value
                else null
            end,
            previous.target_value
        from initiative_metric_value previous
        join metrics_directory directory
            on directory.id = previous.metric_directory_id

        where previous.period_month = :previousPeriod

          /*
           * Не переносим значение, если метрика
           * неприменима для периода, в который
           * выполняется перенос.
           */
          and not exists (
              select 1
              from initiative_metric_assignment assignment
              join metric_applicability_request request
                  on request.initiative_metric_assignment_id = assignment.id
              where assignment.initiative_agent_type_id =
                        previous.initiative_agent_type_id

                and assignment.metric_id =
                        previous.metric_directory_id

                /*
                 * Ограничение уже началось.
                 */
                and request.effective_from_period is not null
                and request.effective_from_period <= :currentPeriod

                and (
                    /*
                     * Завершённый период неприменимости.
                     *
                     * effective_to_period является
                     * exclusive boundary:
                     *
                     * [effective_from_period, effective_to_period)
                     */
                    (
                        request.effective_to_period is not null
                        and :currentPeriod < request.effective_to_period
                    )

                    or

                    /*
                     * Открытый период.
                     *
                     * PENDING:
                     * заявка находится на согласовании.
                     *
                     * NOT_APPLICABLE:
                     * неприменимость согласована без
                     * наступившего окончания периода.
                     */
                    (
                        request.effective_to_period is null
                        and (
                            (
                                assignment.applicability_status = 'PENDING'
                                and request.status = 'PENDING'
                            )
                            or
                            (
                                assignment.applicability_status = 'NOT_APPLICABLE'
                                and request.status = 'APPROVED'
                            )
                        )
                    )
                )
          )

        on conflict (
            initiative_agent_type_id,
            metric_directory_id,
            period_month
        ) do update
        set metric_value = excluded.metric_value,
            target_value = excluded.target_value
        where initiative_metric_value.metric_value is null
          and initiative_metric_value.target_value is null
          and (
              excluded.metric_value is not null
              or excluded.target_value is not null
          )
    """,
)
fun copyValuesToNextPeriod(
    @Param("previousPeriod")
    previousPeriod: LocalDate,

    @Param("currentPeriod")
    currentPeriod: LocalDate,
): Int

```
