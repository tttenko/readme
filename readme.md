```java

@Service
@RequiredArgsConstructor
public class CalendarRangeService {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private static final String STATUS_WORK = "рабочий день";
    private static final String STATUS_NONWORK = "не рабочий день";

    private final CalendDateService prodCalendDateService; // существующий батч+кеш
    private final WorkWindowPolicy workWindowPolicy;       // политика окна

    public List<CalendarRangeItemDto> buildRange(
            String date,
            int numdays,
            boolean isforward,
            String daysClassificationRaw
    ) {
        LocalDate start = parseStart(date);
        validateNumdays(numdays);

        DaysClassification classification = DaysClassification.parseOrDefault(daysClassificationRaw);
        int step = step(isforward);

        List<LocalDate> requestedRange = switch (classification) {
            case ALL -> buildAllDates(start, numdays, step);
            case WORK -> buildWorkDates(start, numdays, step);
        };

        // Грузим МД ровно под итоговый диапазон (батчем). Для WORK это ок, т.к. кеш уже прогрет окнами.
        Map<String, CalendarDateDto> mdByShort = loadFromMdAsDto(requestedRange);

        return toSortedRangeResponse(requestedRange, mdByShort);
    }

    // -------------------- ALL --------------------

    private List<LocalDate> buildAllDates(LocalDate start, int numdays, int step) {
        LocalDate end = start.plusDays((long) step * (numdays - 1L));
        return inclusiveRange(start, end); // всегда по возрастанию
    }

    // -------------------- WORK (оконная загрузка) --------------------

    private List<LocalDate> buildWorkDates(LocalDate start, int numdays, int step) {
        int windowSize = workWindowPolicy.initialWindowSize(numdays);

        while (windowSize <= workWindowPolicy.maxWindowSize()) {
            List<LocalDate> windowDates = generateWindow(start, windowSize, step);
            Map<String, CalendarDateDto> windowMd = loadFromMdAsDto(windowDates);

            LocalDate end = findEndWhenWorkDaysReached(windowDates, windowMd, numdays);
            if (end != null) {
                return inclusiveRange(start, end); // итоговый непрерывный диапазон
            }

            int next = workWindowPolicy.nextWindowSize(windowSize);
            if (next <= windowSize) {
                throw new IllegalStateException("WorkWindowPolicy must increase window size");
            }
            windowSize = next;
        }

        throw new IllegalStateException("Unable to resolve work-range: exceeded max window size");
    }

    private LocalDate findEndWhenWorkDaysReached(
            List<LocalDate> orderedWindowDates,
            Map<String, CalendarDateDto> mdByShort,
            int targetWorkDays
    ) {
        int workCount = 0;

        for (LocalDate d : orderedWindowDates) {
            CalendarDateDto dto = requireDto(d, mdByShort);
            if (isWork(dto.getDateType())) {
                workCount++;
            }
            if (workCount == targetWorkDays) {
                return d;
            }
        }
        return null;
    }

    // -------------------- Response mapping (НОВЫЙ DTO) --------------------

    /**
     * Спека требует:
     * - непрерывный список дат
     * - сортировка по возрастанию (даже при isforward=false)
     * - поля: id, date(dd.MM.yyyy), dateType, dailyStatus
     */
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
                    // ✅ date по спеке: dd.MM.yyyy
                    .date(mdDto.getDateShort())
                    .dateType(mdDto.getDateType())
                    .dailyStatus(isWork(mdDto.getDateType()) ? STATUS_WORK : STATUS_NONWORK)
                    .build());
        }
        return result;
    }

    private boolean isWork(String dateType) {
        return "1".equals(dateType) || "2".equals(dateType);
    }

    // -------------------- MD loading (батч+кеш) --------------------

    /**
     * В МД ищем по item.slug = dd.MM.yyyy (см. пример запроса),
     * поэтому ключи — dateShort.
     */
    private Map<String, CalendarDateDto> loadFromMdAsDto(List<LocalDate> dates) {
        List<String> keys = dates.stream()
                .map(this::formatShort)
                .distinct()
                .toList();

        List<CalendarDateDto> fromMd = prodCalendDateService.searchByDates(keys);

        return fromMd.stream()
                .filter(d -> d.getDateShort() != null)
                .collect(Collectors.toMap(
                        CalendarDateDto::getDateShort,
                        d -> d,
                        (a, b) -> a
                ));
    }

    private CalendarDateDto requireDto(LocalDate date, Map<String, CalendarDateDto> mdByShort) {
        String key = formatShort(date);
        CalendarDateDto dto = mdByShort.get(key);
        if (dto == null) {
            throw new IllegalStateException("No calendar data from MD for date: " + key);
        }
        return dto;
    }

    // -------------------- Date helpers --------------------

    private LocalDate parseStart(String date) {
        try {
            return LocalDate.parse(date, DDMMYYYY);
        } catch (DateTimeParseException e) {
            throw new IllegalArgumentException("date must be in format dd.MM.yyyy");
        }
    }

    private void validateNumdays(int numdays) {
        if (numdays < 1) {
            throw new IllegalArgumentException("numdays must be >= 1");
        }
    }

    private int step(boolean isforward) {
        return isforward ? 1 : -1;
    }

    private String formatShort(LocalDate d) {
        return d.format(DDMMYYYY);
    }

    /**
     * Непрерывный диапазон между a и b включительно.
     * Возвращаем всегда по календарному возрастанию.
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
     * Окно: start, start+step, ... (в направлении step).
     * Порядок здесь важен для подсчёта рабочих дней.
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
