```java
val model = mutableMapOf(
    METRIC_NAME to metricName,
    INITIATIVE_NAME to initiativeName,
    SUPPORT_EMAIL to emailProperties.emailLinkProperties.pultSupportBox,
    LINK to PultLinksHelper.buildLinkToInitiative(
        emailProperties.emailLinkProperties.linkToPortalShort,
        initiativeId
    )
)
```
