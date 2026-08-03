```java
"deviations": {
  "type": "array",
  "maxItems": 2147483647,
  "description": "Список отклонений по инициативе",
  "items": {
    "$ref": "#/definitions/InitiativeDeviationResponse"
  }
},

"InitiativeDeviationResponse": {
  "type": "object",
  "properties": {
    "code": {
      "type": "string",
      "pattern": "^.*$",
      "maxLength": 255,
      "description": "Код отклонения",
      "example": "STAGE_DEADLINE_EXPIRED"
    },
    "priority": {
      "type": "integer",
      "minimum": -2147483648,
      "maximum": 2147483647,
      "description": "Приоритет отображения",
      "example": 1
    },
    "title": {
      "type": "string",
      "pattern": "^.*$",
      "maxLength": 255,
      "description": "Заголовок отклонения",
      "example": "Нарушен дедлайн этапа"
    },
    "description": {
      "type": "string",
      "pattern": "^(.|\\n)*$",
      "maxLength": 1000,
      "description": "Описание отклонения"
    }
  },
  "additionalProperties": false
}
  ```
