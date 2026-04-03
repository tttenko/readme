```java
@PreAuthorize("""
    principal instanceof T(ru.sber.cs.supplier.portal.authorization.user.AuthorizedUser)
    and principal.getUserType() == T(ru.sber.cs.supplier.portal.authorization.user.UserType).SUPPLIER
    """)
```
