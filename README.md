```java
contactEmails = entity.agentContact.mapNotNull { agentContact ->
    if (agentContact.userId != null) {
        userEmailsById[agentContact.userId]
    } else {
        agentContact.contact?.email
    }
}.joinToString(";"),
```
