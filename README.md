```java

@JsonIgnoreProperties(ignoreUnknown = true)
data class SearchIssueFieldsDto(
    val summary: String? = null,
    val description: String? = null,
    val issuetype: SearchIssueTypeDto? = null,

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val labels: List<String> = emptyList(),

    val customfield_29205: String? = null,

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val customfield_30000: List<String> = emptyList(),

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val customfield_30001: List<String> = emptyList(),

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val customfield_30002: List<String> = emptyList(),

    val customfield_34300: String? = null,
    val customfield_30401: String? = null,
    val status: SearchIssueStatusDto? = null,

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val issuelinks: List<SearchIssueLinkDto> = emptyList(),

    val reporter: SearchIssueUserInfoDto? = null,
    val assignee: SearchIssueUserInfoDto? = null,
    val customfield_29202: SearchIssueUserInfoDto? = null,

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val customfield_29203: List<SearchIssueUserInfoDto> = emptyList(),

    val customfield_16700: String? = null,
    val customfield_16701: String? = null,
    val customfield_31304: String? = null,
    val customfield_31305: String? = null,
    val customfield_31306: String? = null,
    val customfield_31307: String? = null,

    @JsonSetter(nulls = Nulls.AS_EMPTY)
    val customfield_15903: List<SearchIssueCheckboxOptionDto> = emptyList(),

    val lastViewed: String? = null,
    val resolutiondate: String? = null,
    val created: String? = null,
    val updated: String? = null,
)
```
