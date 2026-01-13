```java

@Validated включает проверку @RequestParam/@PathVariable и кидает ConstraintViolationException, который ловит наш хендлер.
С @Valid это не гарантируется — часто уходит в дефолтный “Validation failure”
```
