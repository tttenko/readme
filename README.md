```java
@PreAuthorize("""
    #p0 != null
    and #p0.getUserType() == T(ru.sber.cs.supplier.portal.authorization.dto.enums.UserType).SUPPLIER
    """)
```
