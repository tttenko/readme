```java
serviceId: CI07150460_csportalControlVehcl
version: "1.0"
description: Управление СТС

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
      - id: sts_uuid
        description: UUID СТС
        hidden: true
        sendToAudit: true
      - id: contract_uuid
        description: UUID договора
        hidden: true
        sendToAudit: true
      - id: tb_id
        description: Код ТБ
        hidden: false
        sendToAudit: true
      - id: vehicle_num
        description: Номер ТС
        hidden: true
        sendToAudit: true
      - id: vehicle_name
        description: Марка ТС
        hidden: true
        sendToAudit: true
      - id: comment
        description: Примечание
        hidden: false
        sendToAudit: false
      - id: status
        description: Статус СТС
        hidden: false
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
      - id: sts_uuid
        description: UUID СТС
        hidden: true
        sendToAudit: true
      - id: contract_uuid
        description: UUID договора
        hidden: true
        sendToAudit: true
    changedFields:
      - id: tb_id
        description: Код ТБ
        hidden: false
        sendToAudit: false
      - id: vehicle_num
        description: Номер ТС
        hidden: false
        sendToAudit: false
      - id: vehicle_name
        description: Марка ТС
        hidden: false
        sendToAudit: false
      - id: comment
        description: Примечание
        hidden: false
        sendToAudit: false

  - id: STS_STATUS_CHANGE
    description: Изменение статуса СТС
    successful: true
    audit:
      mode: reliability
      code: AUD-30
    ui:
      type: edit
    parameters:
      - id: sts_uuid
        description: UUID СТС
        hidden: true
        sendToAudit: true
      - id: contract_uuid
        description: UUID договора
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
      - id: sts_uuid
        description: UUID СТС
        hidden: true
        sendToAudit: true
      - id: contract_uuid
        description: UUID договора
        hidden: true
        sendToAudit: true
      - id: vehicle_num
        description: Номер ТС
        hidden: true
        sendToAudit: true
      - id: vehicle_name
        description: Марка ТС
        hidden: true
        sendToAudit: true
      - id: status
        description: Статус СТС
        hidden: false
        sendToAudit: true
```
