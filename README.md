```java
app:
  tracker:
    kafka:
      topic:
        scheme: ${APP_TRACKER_KAFKA_TOPIC_SCHEME:tracker_scheme}
        history: ${APP_TRACKER_KAFKA_TOPIC_HISTORY:tracker_history}
        additional: ${APP_TRACKER_KAFKA_TOPIC_ADDITIONAL:tracker_additional}
      send:
        active: ${APP_TRACKER_KAFKA_SEND_ACTIVE:true}
      scheme:
        publish-on-startup: ${APP_TRACKER_KAFKA_SCHEME_PUBLISH_ON_STARTUP:true}
```
