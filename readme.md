```java

@Override
public List<LocalDate> calculate(LocalDate start, int numDays, int step) {
    int windowSize = windowPolicy.initialWindowSize(numDays);

    // Кэш уже загруженных дат, чтобы при росте окна не дергать МД повторно
    Map<LocalDate, CalendarDateDto> mdIndex = new HashMap<>();

    while (windowSize <= windowPolicy.maxWindowSize()) {

        List<LocalDate> windowDates =
                calendarRangeHelpers.generateWindow(start, windowSize, step);

        // ВАЖНО: грузим только даты, которых ещё нет в mdIndex
        List<LocalDate> missing = windowDates.stream()
                .filter(d -> !mdIndex.containsKey(d))
                .toList();

        if (!missing.isEmpty()) {
            mdIndex.putAll(dataProvider.loadByDates(missing));
        }

        Optional<LocalDate> endOpt = findEndDate(windowDates, mdIndex, numDays);
        if (endOpt.isPresent()) {
            return calendarRangeHelpers.inclusiveRange(start, endOpt.get());
        }

        int next = windowPolicy.nextWindowSize(windowSize, numDays);
        if (next <= windowSize) {
            throw new IllegalStateException(
                    "WorkWindowPolicy must increase window size: current=" + windowSize + ", next=" + next
            );
        }
        windowSize = next;
    }

    // При текущих гарантиях это "такого быть не должно"
    throw new IllegalStateException(
            "Invariant violated: work-range end not found. start=" + start +
            ", numDays=" + numDays +
            ", step=" + step +
            ", maxWindow=" + windowPolicy.maxWindowSize()
    );
}

private Optional<LocalDate> findEndDate(
        List<LocalDate> orderedDates,
        Map<LocalDate, CalendarDateDto> md,
        int targetWorkDays
) {
    if (targetWorkDays <= 0) {
        return Optional.empty();
    }

    int workCount = 0;

    for (LocalDate d : orderedDates) {
        CalendarDateDto dto = requireDto(d, md);

        if (calendarRangeHelpers.workdayClassifier(dto.getDateType())) {
            workCount++;
            if (workCount == targetWorkDays) {
                return Optional.of(d);
            }
        }
    }

    return Optional.empty();
}

private CalendarDateDto requireDto(
        LocalDate date,
        Map<LocalDate, CalendarDateDto> md
) {
    CalendarDateDto dto = md.get(date);
    if (dto == null) {
        throw new MissingCalendarDataException(date);
    }
    return dto;
}
```
