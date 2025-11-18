```java

 private String anyValidDateForNovat0(GetItemsSearchResponse response) {
    // Преобразуем ответ МД в список NdsFullDto так же, как в боевом коде
    List<NdsFullDto> items =
            BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper);

    NdsFullDto novat0 = items.stream()
            .filter(e -> "NOVAT".equals(e.getCode()) && "0".equals(e.getRate()))
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("NOVAT with rate=0 not found in fixture"));

    ZonedDateTime start = novat0.getRateDateStartZoned();
    ZonedDateTime end   = novat0.getRateDateEndZoned();

    // Берём дату заведомо внутри интервала действия ставки
    ZonedDateTime target;
    if (start != null && end != null) {
        target = start.plusSeconds(1);
        if (target.isAfter(end)) {
            // на случай, если start == end
            target = start;
        }
    } else if (start != null) {
        target = start.plusSeconds(1);
    } else if (end != null) {
        target = end.minusSeconds(1);
    } else {
        // если оба null — ставка «всегда активна», можно взять любое время
        target = ZonedDateTime.now();
    }

    return target.format(NDS_DATE_FORMATTER);
}


@Autowired
private NdsMapper ndsMapper;

private static final DateTimeFormatter NDS_DATE_FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

        

```
