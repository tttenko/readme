```java

/**
 * Обрабатывает заявки с наступившим resumePeriod.
 *
 * Сначала закрывает период неприменимости и скрывает заявку из очереди Офиса,
 * после чего возвращает связанную метрику в ACTIVE.
 */
@Transactional
fun checkAndProcessExpiredMetrics() {
    // 1. Обновляем request, пока assignment ещё PENDING / NOT_APPLICABLE
    metricApplicabilityRequestRepository.updateRequests()

    // 2. Возвращаем assignment в ACTIVE
    metricApplicabilityRequestRepository.updateAssignmentsToActive()

    // 3. Получаем обработанные scheduler заявки
    val updatedRequests =
        metricApplicabilityRequestRepository.findUpdatedRequests()

    // 4. Отправляем уведомления
    updatedRequests.forEach { sendNotifications(it) }
}


```
