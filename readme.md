```java

@Component
@RequiredArgsConstructor
public class CalendarDateMdIndexLoader {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final CalendarDateService prodCalendDateService; // существующий batch + cache

    /**
     * Загружает календарные DTO из МД батчем (по item.slug = dd.MM.yyyy)
     * и индексирует их по dateShort (dd.MM.yyyy) для O(1) доступа.
     */
    public Map<String, CalendarDateDto> loadIndexByDateShort(List<LocalDate> dates) {
        List<String> keys = toDistinctShortKeys(dates);
        List<CalendarDateDto> dtos = prodCalendDateService.searchByDates(keys);
        return indexByDateShort(dtos);
    }

    /** Ключ для МД (item.slug) и для индекса. */
    public String toShortKey(LocalDate date) {
        return date.format(DDMMYYYY);
    }

    private List<String> toDistinctShortKeys(List<LocalDate> dates) {
        return dates.stream()
            .map(this::toShortKey)
            .distinct()
            .toList();
    }

    private Map<String, CalendarDateDto> indexByDateShort(List<CalendarDateDto> dtos) {
        return dtos.stream()
            .filter(d -> d.getDateShort() != null)
            .collect(Collectors.toMap(
                CalendarDateDto::getDateShort,
                Function.identity(),
                (a, b) -> a // если внезапно дубль — берём первый
            ));
    }
}

@Service
@RequiredArgsConstructor
public class CalendarRangeService {

    private static final String STATUS_WORK = "рабочий день";
    private static final String STATUS_NON_WORK = "не рабочий день";

    private final CalendarDateMdIndexLoader mdLoader;
    private final DefaultWorkWindowPolicy workWindowPolicy;

    /**
     * ВАЖНО: здесь предполагаем, что контроллер уже провалидировал входные параметры:
     * - date в формате dd.MM.yyyy и распарсен в LocalDate
     * - numDays >= 1
     * - isForward / daysClassification имеют дефолты
     */
    public List<CalendarRangeItemDto> buildRange(
        LocalDate start,
        int numDays,
        boolean isForward,
        String daysClassificationRaw
    ) {
        DaysClassification classification = DaysClassification.parseOrDefault(daysClassificationRaw);
        int step = step(isForward);

        List<LocalDate> requestedRange = switch (classification) {
            case ALL -> buildAllDates(start, numDays, step);
            case WORK -> buildWorkDates(start, numDays, step);
        };

        // финально грузим МД ровно на итоговый диапазон (батчем)
        Map<String, CalendarDateDto> mdByShort = mdLoader.loadIndexByDateShort(requestedRange);
        return toSortedRangeResponse(requestedRange, mdByShort);
    }

    // ---------------- ALL ----------------

    private List<LocalDate> buildAllDates(LocalDate start, int numDays, int step) {
        LocalDate end = start.plusDays((long) step * (numDays - 1L));
        return inclusiveRange(start, end); // всегда по возрастанию
    }

    // ---------------- WORK (оконная загрузка) ----------------

    private List<LocalDate> buildWorkDates(LocalDate start, int numDays, int step) {
        int windowSize = workWindowPolicy.initialWindowSize(numDays);

        while (windowSize <= workWindowPolicy.maxWindowSize()) {
            List<LocalDate> windowDates = generateWindow(start, windowSize, step);
            Map<String, CalendarDateDto> windowMd = mdLoader.loadIndexByDateShort(windowDates);

            Optional<LocalDate> endDate = tryFindEndDateByWorkDays(windowDates, windowMd, numDays);
            if (endDate.isPresent()) {
                return inclusiveRange(start, endDate.get()); // итоговый непрерывный диапазон
            }

            int next = workWindowPolicy.nextWindowSize(windowSize);
            if (next <= windowSize) {
                throw new IllegalStateException("WorkWindowPolicy must increase window size");
            }
            windowSize = next;
        }

        throw new IllegalStateException("Unable to resolve work-range: exceeded max window size");
    }

    private Optional<LocalDate> tryFindEndDateByWorkDays(
        List<LocalDate> orderedWindowDates,
        Map<String, CalendarDateDto> mdByShort,
        int targetWorkDays
    ) {
        if (targetWorkDays <= 0) {
            return Optional.empty();
        }

        int workCount = 0;
        for (LocalDate d : orderedWindowDates) {
            CalendarDateDto dto = requireDto(d, mdByShort);

            if (isWork(dto.getDateType())) {
                workCount++;
            }
            if (workCount == targetWorkDays) {
                return Optional.of(d);
            }
        }
        return Optional.empty();
    }

    // ---------------- RESPONSE ----------------

    private List<CalendarRangeItemDto> toSortedRangeResponse(
        List<LocalDate> rangeDates,
        Map<String, CalendarDateDto> mdByShort
    ) {
        List<LocalDate> sorted = rangeDates.stream().sorted().toList();

        List<CalendarRangeItemDto> result = new ArrayList<>(sorted.size());
        for (LocalDate d : sorted) {
            CalendarDateDto mdDto = requireDto(d, mdByShort);

            result.add(CalendarRangeItemDto.builder()
                .id(mdDto.getId())
                .date(mdDto.getDateShort())                 // строго dd.MM.yyyy по спеке
                .dateType(mdDto.getDateType())
                .dailyStatus(isWork(mdDto.getDateType()) ? STATUS_WORK : STATUS_NON_WORK)
                .build());
        }
        return result;
    }

    // ---------------- UTILS ----------------

    private CalendarDateDto requireDto(LocalDate date, Map<String, CalendarDateDto> mdByShort) {
        String key = mdLoader.toShortKey(date);
        CalendarDateDto dto = mdByShort.get(key);

        if (dto == null) {
            throw new IllegalStateException("No calendar data from MD for date: " + key);
        }
        return dto;
    }

    private boolean isWork(String dateType) {
        return "1".equals(dateType) || "2".equals(dateType);
    }

    private int step(boolean isForward) {
        return isForward ? 1 : -1;
    }

    /**
     * Непрерывный диапазон между a и b включительно, всегда по возрастанию.
     * (Это важно: даже если isForward=false, ответ должен быть отсортирован по возрастанию.)
     */
    private List<LocalDate> inclusiveRange(LocalDate a, LocalDate b) {
        LocalDate start = a.isBefore(b) ? a : b;
        LocalDate end = a.isBefore(b) ? b : a;

        List<LocalDate> out = new ArrayList<>();
        LocalDate cur = start;
        while (!cur.isAfter(end)) {
            out.add(cur);
            cur = cur.plusDays(1);
        }
        return out;
    }

    /**
     * Окно дат: start, start+step, ... (windowSize штук).
     * Порядок тут важен — по нему считаем рабочие дни.
     */
    private List<LocalDate> generateWindow(LocalDate start, int windowSize, int step) {
        List<LocalDate> out = new ArrayList<>(windowSize);
        LocalDate cur = start;

        for (int i = 0; i < windowSize; i++) {
            out.add(cur);
            cur = cur.plusDays(step);
        }
        return out;
    }
}
```
