```java

public interface FxEnvelope {
  UUID getRquid();
  LocalDateTime getRqtm();
  List<FxRateXmlDto> getFxRates();
}

@Override public UUID getRquid() { return rquid; }
  @Override public LocalDateTime getRqtm() { return rqtm; }
  @Override public List<FxRateXmlDto> getFxRates() { return fxRates; }
```
