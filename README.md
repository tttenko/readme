```java
code: status_sts

nodes:
  - id: DRAFT
    start: true
    text: Черновик
    edges:
      - target: TO_APPROVE_IN
        id: IN-APPROVE
        validator: toapprovein
        base: true
      - target: TO_APPROVE_OUT
        id: TO-DELETE
        validator: toapproveout

  - id: TO_APPROVE_IN
    text: На согласовании включения
    edges:
      - target: APPROVE
        id: APPROVE
        validator: approvein
        base: true
      - target: DRAFT
        validator: rejectin

  - id: TO_APPROVE_OUT
    text: На согласовании удаления
    edges:
      - target: DELETE
        validator: approveout
        base: true
      - target: APPROVE
        validator: rejectout

  - id: APPROVE
    text: Включен в договор
    edges:
      - target: TO_APPROVE_OUT
        validator: toapproveout

  - id: DELETE
    text: Удален
    end: true
```
