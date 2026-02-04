```java

@Component
public class DefaultWorkWindowPolicy implements WorkWindowPolicy {
    private static final int MIN_WINDOW = 30;
    private static final int MAX_WINDOW = 4000;

    @Override
    public int initialWindowSize(int numDays) {
        return Math.min(Math.max(numDays * 2 + 20, MIN_WINDOW), MAX_WINDOW);
    }

    @Override
    public int nextWindowSize(int current, int numDays) {
        // экспоненциально, но ограничено
        int grown = current * 2;
        int linear = current + Math.max(50, numDays);
        return Math.min(Math.max(grown, linear), MAX_WINDOW);
    }

    @Override
    public int maxWindowSize() { return MAX_WINDOW; }
}

public interface RangeCalculator {
    List<LocalDate> calculate(LocalDate start, int numDays, int step);
}

@Component
public class AllRangeCalculator implements RangeCalculator {
    @Override
    public List<LocalDate> calculate(LocalDate start, int numDays, int step) {
        LocalDate end = start.plusDays((long) step * (numDays - 1));
        return inclusiveAscending(start, end);
    }
}

@Component
@RequiredArgsConstructor
public class WorkRangeCalculator {

    private final WorkWindowPolicy windowPolicy;
    private final CalendarDataProvider dataProvider;
    private final WorkdayClassifier classifier; // isWork(dateType)

    public List<LocalDate> calculate(LocalDate start, int numDays, int step) {
        int windowSize = windowPolicy.initialWindowSize(numDays);

        // накапливаем индекс постепенно
        Map<LocalDate, CalendarDateDto> mdIndex = new HashMap<>();
        List<LocalDate> windowDates = new ArrayList<>();

        while (windowSize <= windowPolicy.maxWindowSize()) {
            windowDates = generateWindow(start, windowSize, step);

            // грузим только недостающие даты
            List<LocalDate> missing = windowDates.stream()
                    .filter(d -> !mdIndex.containsKey(d))
                    .toList();

            if (!missing.isEmpty()) {
                mdIndex.putAll(dataProvider.loadByDates(missing));
            }

            LocalDate end = findEndDate(windowDates, mdIndex, numDays);
            if (end != null) {
                return inclusiveAscending(start, end);
            }

            int next = windowPolicy.nextWindowSize(windowSize, numDays);
            if (next <= windowSize) {
                throw new IllegalStateException("WorkWindowPolicy must increase window size");
            }
            windowSize = next;
        }

        throw new IllegalStateException("Unable to resolve work-range: exceeded max window size");
    }

    private LocalDate findEndDate(List<LocalDate> orderedDates,
                                 Map<LocalDate, CalendarDateDto> md,
                                 int targetWorkDays) {
        int workCount = 0;
        for (LocalDate d : orderedDates) {
            CalendarDateDto dto = requireDto(d, md);
            if (classifier.isWork(dto.getDateType())) {
                workCount++;
                if (workCount == targetWorkDays) return d;
            }
        }
        return null;
    }

    private CalendarDateDto requireDto(LocalDate date, Map<LocalDate, CalendarDateDto> md) {
        CalendarDateDto dto = md.get(date);
        if (dto == null) throw new MissingCalendarDataException(date);
        return dto;
    }

    private static List<LocalDate> inclusiveAscending(LocalDate a, LocalDate b) { ... }
    private static List<LocalDate> generateWindow(LocalDate start, int windowSize, int step) { ... }
}

@Component
@RequiredArgsConstructor
public class CalendarRangeMapper {

    private final WorkdayClassifier classifier;

    public List<CalendarRangeItemDto> map(List<LocalDate> ascendingDates,
                                         Map<LocalDate, CalendarDateDto> md) {
        List<CalendarRangeItemDto> result = new ArrayList<>(ascendingDates.size());

        for (LocalDate d : ascendingDates) {
            CalendarDateDto dto = md.get(d);
            if (dto == null) throw new MissingCalendarDataException(d);

            result.add(CalendarRangeItemDto.builder()
                    .id(dto.getId())
                    .date(dto.getDateShort())
                    .dateType(dto.getDateType())
                    .dailyStatus(classifier.isWork(dto.getDateType())
                            ? "Рабочий день" : "Не рабочий день")
                    .build());
        }
        return result;
    }
}

@Component
public class WorkdayClassifier {
    public boolean isWork(String dateType) {
        return "1".equals(dateType) || "2".equals(dateType);
    }
}

@Service
@RequiredArgsConstructor
public class CalendarRangeService {

    private final CalendarDataProvider dataProvider;
    private final AllRangeCalculator allCalc;
    private final WorkRangeCalculator workCalc;
    private final CalendarRangeMapper mapper;

    public List<CalendarRangeItemDto> buildRange(LocalDate start, int numDays,
                                                 boolean isForward, String raw) {
        if (numDays < 1) throw new IllegalArgumentException("numDays must be >= 1");

        DaysClassification cls = DaysClassification.parseOrDefault(raw);
        int step = isForward ? 1 : -1;

        List<LocalDate> dates = switch (cls) {
            case ALL -> allCalc.calculate(start, numDays, step);
            case WORK -> workCalc.calculate(start, numDays, step);
        };

        Map<LocalDate, CalendarDateDto> md = dataProvider.loadByDates(dates);
        return mapper.map(dates, md); // dates уже ascending
    }
}


```
