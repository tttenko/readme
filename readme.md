```java
 repo.upsert(
          e.getRquid(), e.getRqtm(),
          e.getFxRateSubType(),
          e.getCode1(), e.getIsoNum1(),
          e.getCode2(), e.getIsoNum2(),
          e.getUseDate(),
          e.getLotSize(), e.getValue()
          // , e.getIsPublic() если нужно
      );
```
