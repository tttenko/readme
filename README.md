```java

return contactRepository.findAllByEmailIn(emails)
    .filter { !it.email.isNullOrBlank() }
    .associateFirstBy { it.email!! }
    .toMutableMap()

```
