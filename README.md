```java
events:
  - id: STS_CREATE
    description: Создание СТС
    successful: true
    audit:
      mode: reliability
      code: AUD-30
    ui:
      type: create
    parameters:
      - id: vehicle_num
        description: Номер ТС
        hidden: true
        sendToAudit: true
      - id: vehicle_name
        description: Марка ТС
        hidden: true
        sendToAudit: true

  - id: STS_UPDATE
    description: Изменение СТС
    successful: true
    audit:
      mode: reliability
      code: AUD-30
    ui:
      type: edit
    parameters:
      - id: vehicle_num
        description: Номер ТС
        hidden: true
        sendToAudit: true
      - id: vehicle_name
        description: Марка ТС
        hidden: true
        sendToAudit: true
    changedFields:
      - id: status
        description: Статус СТС
        hidden: false
        sendToAudit: true

  - id: STS_DELETE
    description: Удаление СТС
    successful: true
    audit:
      mode: reliability
      code: AUD-30
    ui:
      type: delete
    parameters:
      - id: vehicle_num
        description: Номер ТС
        hidden: true
        sendToAudit: true
      - id: vehicle_name
        description: Марка ТС
        hidden: true
        sendToAudit: true
```
