```java
every { enablerNameNormalizer.normalize("GIGACHAT") } returns "gigachat"
    every { enablerNameNormalizer.normalize("DATA_LENS") } returns "data_lens"

every { enablerNameNormalizer.normalize("GIGACHAT") } returns "gigachat"
    every { enablerNameNormalizer.normalize("  ") } returns null
```
