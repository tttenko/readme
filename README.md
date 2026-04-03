```java
@PreAuthorize("""
    #principal != null
    and #principal.getUserType() == T(ru.sber.cs.supplier.portal.authorization.dto.enums.UserType).SUPPLIER
    """)
```
