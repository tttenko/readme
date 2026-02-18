```java/**
@Service
@RequiredArgsConstructor
@Log4j2
public class FxRateIngestService {

    private final FxRateRepository repo;
    private final FxRateEntityMapper fxRateEntityMapper;

    /** Обрабатывает входящее сообщение {@link PutEodPriceNfDto} и сохраняет курсы в БД. */
    @Transactional
    public void ingest(PutEodPriceNfDto message) {
        if (message == null) {
            throw new IllegalArgumentException(
                "Входящее сообщение PutEodPriceNfDto равно null, "
                    + "ожидается распарсенный XML PutEODPriceNF."
            );
        }

        String requestUid = message.rquid;
        LocalDateTime requestTime = message.rqtm;
        List<FxRateXmlDto> rates = message.fxRates;

        if (requestUid == null || requestUid.isBlank()) {
            throw new InvalidCurrencyRateXmlException(
                "Некорректный входной XML: отсутствует обязательный идентификатор запроса RqUID "
                    + "(элемент PutEODPriceNF/RqUID)."
            );
        }

        if (requestTime == null) {
            throw new InvalidCurrencyRateXmlException(String.format(
                "Некорректный входной XML: отсутствует обязательное время запроса RqTm "
                    + "(элемент PutEODPriceNF/RqTm). RqUID=%s.",
                requestUid
            ));
        }

        if (rates == null) {
            throw new InvalidCurrencyRateXmlException(String.format(
                "Некорректный входной XML: отсутствуют курсы FXRates. RqUID=%s.",
                requestUid
            ));
        }

        for (FxRateXmlDto fxRateXmlDto : rates) {
            FxRateEntity entity = fxRateEntityMapper.toEntity(requestUid, requestTime, fxRateXmlDto);

            ExchangeRateUpsertParams rateParams = new ExchangeRateUpsertParams(
                entity.getRequestUid(),
                entity.getRequestTime(),
                entity.getFxRateSubType(),
                entity.getFromCurrencyCode(),
                entity.getFromCurrencyIsoNum(),
                entity.getToCurrencyCode(),
                entity.getToCurrencyIsoNum(),
                entity.getUseDate(),
                entity.getLotSize(),
                entity.getValue()
            );

            repo.upsert(rateParams);
        }
    }
}
```
