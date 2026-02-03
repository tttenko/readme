```java

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Элемент диапазона календаря")
public class CalendarRangeItemDto implements Serializable {

    @Schema(description = "UUID записи")
    private String id;

    @Schema(description = "Дата в формате dd.MM.yyyy")
    private String date;

    @Schema(description = "Код типа дня")
    private String dateType;

    @Schema(description = "Статус дня: рабочий день / не рабочий день")
    private String dailyStatus;
}

public enum DaysClassification {
    ALL("all"),
    WORK("work");

    private final String wire;

    DaysClassification(String wire) {
        this.wire = wire;
    }

    public static DaysClassification parseOrDefault(String raw) {
        if (raw == null || raw.isBlank()) return ALL;
        String v = raw.trim().toLowerCase();

        for (DaysClassification dc : values()) {
            if (dc.wire.equals(v)) return dc;
        }
        throw new IllegalArgumentException("daysClassification must be one of: all, work");
    }
}

@Component
public class DefaultWorkWindowPolicy implements WorkWindowPolicy {

    private static final int MIN_WINDOW = 30;
    private static final int GROW_STEP = 50;
    private static final int MAX_WINDOW = 4000;

    @Override
    public int initialWindowSize(int numdays) {
        int w = numdays * 2 + 20;
        if (w < MIN_WINDOW) w = MIN_WINDOW;
        if (w > MAX_WINDOW) w = MAX_WINDOW;
        return w;
    }

    @Override
    public int nextWindowSize(int currentWindowSize) {
        return Math.min(currentWindowSize + GROW_STEP, MAX_WINDOW);
    }

    @Override
    public int maxWindowSize() {
        return MAX_WINDOW;
    }
}

@Service
@RequiredArgsConstructor
public class CalendarRangeService {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private static final String STATUS_WORK = "рабочий день";
    private static final String STATUS_NONWORK = "не рабочий день";

    private final CalendDateService prodCalendDateService; // ваш существующий батч+кеш сервис
    private final WorkWindowPolicy workWindowPolicy;

    public List<CalendarDateDto> buildRange(
            String date,
            int numdays,
            boolean isforward,
            String daysClassificationRaw
    ) {
        LocalDate start = parseStart(date);
        validateNumdays(numdays);

        DaysClassification classification = DaysClassification.parseOrDefault(daysClassificationRaw);
        int step = step(isforward);

        List<LocalDate> rangeDates = switch (classification) {
            case ALL -> buildAllDates(start, numdays, step);
            case WORK -> buildWorkDates(start, numdays, step);
        };

        Map<String, CalendarDateDto> byShort = loadFromMdAsDto(rangeDates);

        List<CalendarDateDto> sorted = toSortedResponse(rangeDates, byShort);
        enrichDailyStatus(sorted);

        return sorted;
    }

    // -------------------- ALL --------------------

    private List<LocalDate> buildAllDates(LocalDate start, int numdays, int step) {
        LocalDate end = start.plusDays((long) step * (numdays - 1L));
        return inclusiveRange(start, end);
    }

    // -------------------- WORK (оконная загрузка) --------------------

    private List<LocalDate> buildWorkDates(LocalDate start, int numdays, int step) {
        int windowSize = workWindowPolicy.initialWindowSize(numdays);

        while (windowSize <= workWindowPolicy.maxWindowSize()) {
            List<LocalDate> window = generateWindow(start, windowSize, step);
            Map<String, CalendarDateDto> byShort = loadFromMdAsDto(window);

            LocalDate end = findEndWhenWorkDaysReached(window, byShort, numdays);
            if (end != null) {
                return inclusiveRange(start, end);
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
            Map<String, CalendarDateDto> byShort,
            int targetWorkDays
    ) {
        int workCount = 0;

        for (LocalDate d : orderedWindowDates) {
            CalendarDateDto dto = requireDto(d, byShort);

            if (isWork(dto.getDateType())) {
                workCount++;
            }
            if (workCount == targetWorkDays) {
                return d;
            }
        }
        return null;
    }

    // -------------------- MD loading (батч+кеш) --------------------

    /**
     * ВАЖНО:
     * - В МД по спеке ищем по slug (dd.MM.yyyy), поэтому ключи — dateShort.
     * - searchByDates(list) уже батч+кеш, при расширении окна догрузятся только miss'ы.
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

    private CalendarDateDto requireDto(LocalDate date, Map<String, CalendarDateDto> byShort) {
        String key = formatShort(date);
        CalendarDateDto dto = byShort.get(key);
        if (dto == null) {
            throw new IllegalStateException("No calendar data from MD for date: " + key);
        }
        return dto;
    }

    // -------------------- Response shaping --------------------

    private List<CalendarDateDto> toSortedResponse(
            List<LocalDate> rangeDates,
            Map<String, CalendarDateDto> byShort
    ) {
        List<LocalDate> sorted = rangeDates.stream().sorted().toList();

        List<CalendarDateDto> result = new ArrayList<>(sorted.size());
        for (LocalDate d : sorted) {
            result.add(requireDto(d, byShort));
        }
        return result;
    }

    private void enrichDailyStatus(List<CalendarDateDto> dtos) {
        for (CalendarDateDto dto : dtos) {
            dto.setDailyStatus(isWork(dto.getDateType()) ? STATUS_WORK : STATUS_NONWORK);
        }
    }

    private boolean isWork(String dateType) {
        return "1".equals(dateType) || "2".equals(dateType);
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
     * Возвращаем всегда по календарному возрастанию (по спеке это важно при isforward=false).
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

 @GetMapping("/range")
    ResultObj<List<CalendarDateDto>> calendarRange(
            @RequestParam("date")
            @NotBlank(message = "date не должен быть пустым")
            @Pattern(regexp = DATE_REGEX, message = "Дата должна быть в формате dd.MM.yyyy")
            String date,

            @RequestParam("numdays")
            @Min(value = 1, message = "numdays должен быть >= 1")
            int numdays,

            @RequestParam(value = "isforward", required = false, defaultValue = "true")
            boolean isforward,

            @RequestParam(value = "daysClassification", required = false, defaultValue = "all")
            String daysClassification
    );

    @Override
    public ResultObj<List<CalendarDateDto>> calendarRange(
            String date,
            int numdays,
            boolean isforward,
            String daysClassification
    ) {
        List<CalendarDateDto> items =
                calendarRangeService.buildRange(date, numdays, isforward, daysClassification);

        // ваш стандартный ответ (messages + count + data)
        return getSuccessResponse(items);
    }
```
