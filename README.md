```java
server:
  servlet:
    context-path: ${app.path:/control-vehicle}
  port: ${port:8080}

spring:
  application:
    name: control-vehicle-app
    version: @project.version@

  datasource:
    url: ${db.url}
    username: ${db.username}
    password: ${db.password}

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        default_schema: ${app.db.schema:CONTROL_VEHICLE_APP}
        highlight_sql: true

  liquibase:
    change-log: classpath:database/changelog.xml
    default-schema: ${spring.jpa.properties.hibernate.default_schema}
    enabled: true

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

preliquibase:
  sql-script-references: classpath:/database/preliquibase/postgresql.sql

springdoc:
  swagger-ui:
    enabled: ${app.swagger.enabled:true}
    operationsSorter: method
    tagsSorter: alpha
  show-actuator: true
  api-docs:
    enabled: ${app.swagger.enabled:true}
    version: OPENAPI_3_0
  override-with-generic-response: false
  group-configs:
    - group: ui
      displayName: UI
      pathsToMatch:
        - /ui/v1/**
    - group: actuator
      displayName: Actuator
      pathsToMatch:
        - /actuator/**

management:
  server:
    port: ${app.actuator.port:8080}
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: ${app.actuator.endpoints:*}
  httpexchanges:
    recording:
      enabled: ${app.actuator.http.log:true}

events-history:
  client:
    url: ${app.client.services.app-events-history-service:http://localhost:8081/events-history}
    mode: sync
  metamodel:
    path: classpath:events.yml
    mode: ${app.client.services.events-history.mode:bypass}

tracker:
  scheme:
    paths:
      - classpath:tracker/status-sts.yml

app:
  tracker:
    kafka:
      topic:
        scheme: ${app.tracker.kafka.topic.scheme:tracker_scheme}
        history: ${app.tracker.kafka.topic.history:tracker_history}
        additional: ${app.tracker.kafka.topic.additional:tracker_additional}
      send:
        active: ${app.tracker.kafka.send.active:true}
      scheme:
        publish-on-startup: ${app.tracker.kafka.scheme.publish-on-startup:true}


```
