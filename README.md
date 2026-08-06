```java
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "$ref": "#/definitions/PageSwagger",
  "definitions": {
    "BlockResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": "integer",
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Id блока",
          "example": 21
        },
        "name": {
          "type": "string",
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Название блока",
          "example": "Название блока"
        },
        "shortName": {
          "type": "string",
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Название трайба",
          "example": "Короткое название блока"
        },
        "label": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Метка"
        },
        "code": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "код блока",
          "example": "код"
        },
        "curatorName": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "имя куратора блока",
          "example": "куратора"
        },
        "disabled": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "Признак отключения блока"
        }
      },
      "additionalProperties": false
    },
    "ProgramResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": "integer",
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Id программы",
          "example": 21
        },
        "name": {
          "type": "string",
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Название",
          "example": "Название"
        },
        "fileName": {
          "type": "string",
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Название file",
          "example": "file"
        },
        "code": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "код",
          "example": "код"
        },
        "disabled": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "Признак отключения"
        }
      },
      "additionalProperties": false
    },
    "AiAgentStatusResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Идентификатор агента"
        },
        "name": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Название категории",
          "example": "Общие"
        },
        "ordering": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": -4503599627370496,
          "maximum": 4503599627370496,
          "description": "ordering"
        },
        "code": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 50,
          "description": "code"
        }
      },
      "additionalProperties": false
    },
    "DivisionResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "id"
        },
        "blockId": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "blockId"
        },
        "name": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "name"
        },
        "shortName": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "short name"
        },
        "code": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 50,
          "description": "code"
        },
        "label": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Метка"
        },
        "disabled": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "Признак отключения"
        }
      },
      "additionalProperties": false
    },
    "PlatformDetailResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "id"
        },
        "name": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "name"
        },
        "code": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 250,
          "description": "code"
        },
        "released": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "empty"
        }
      },
      "additionalProperties": false
    },
    "ContactResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "type": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "type",
          "example": "type"
        },
        "contact": {
          "$ref": "#/definitions/ContactDetailResponse"
        },
        "userId": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "user id"
        }
      },
      "additionalProperties": false
    },
    "ContactDetailResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Идентификатор контакта"
        },
        "fio": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "ФИО"
        },
        "email": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "email"
        },
        "invited": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 50,
          "description": "Дата пригляшения"
        }
      },
      "additionalProperties": false
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
    },
    "AiAgentShowcaseResponse": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "id": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Идентификатор агента"
        },
        "program": {
          "$ref": "#/definitions/ProgramResponse"
        },
        "agentDescription": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^(.|\n)*$",
          "maxLength": 1000,
          "description": "agentDescription"
        },
        "agentEffectOptimization": {
          "type": [
            "number",
            "null"
          ],
          "example": 359699.5,
          "minimum": -999999999999.99,
          "maximum": 999999999999.99,
          "description": "agentEffectOptimization"
        },
        "agentEffectRevenue": {
          "type": [
            "number",
            "null"
          ],
          "example": 359699.5,
          "minimum": -999999999999.99,
          "maximum": 999999999999.99,
          "description": "agentEffectOptimization"
        },
        "agentId": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "agentId"
        },
        "agentName": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "agentName"
        },
        "agentProblem": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^(.|\\n)*$",
          "maxLength": 1000,
          "description": "agentProblem"
        },
        "agentSolution": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^(.|\\n)*$",
          "maxLength": 1000,
          "description": "agentSolution"
        },
        "agentStatus": {
          "$ref": "#/definitions/AiAgentStatusResponse"
        },
        "contacts": {
          "type": "array",
          "maxItems": 2147483647,
          "items": {
            "anyOf": [
              {
                "$ref": "#/definitions/ContactResponse"
              },
              {
                "type": "null"
              }
            ]
          }
        },
        "implementedPlatforms": {
          "type": "array",
          "maxItems": 2147483647,
          "items": {
            "$ref": "#/definitions/PlatformDetailResponse"
          }
        },
        "division": {
          "$ref": "#/definitions/DivisionResponse"
        },
        "block": {
          "$ref": "#/definitions/BlockResponse"
        },
        "agentInitiativeType": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Тип инициативы"
        },
        "deadlineExpired": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "Cтатус внедрения просрочен"
        },
        "terbankId": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "terbank id"
        },
        "hasMetricsValue": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "Признак наличия режимов работы инициативы"
        },
        "overdueDays": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": -4503599627370496,
          "maximum": 4503599627370496,
          "description": "Кол-во дней просрочки по статусу"
        },
        "deviations": {
          "type": "array",
          "maxItems": 2147483647,
          "description": "Список отклонений по инициативе",
          "items": {
            "$ref": "#/definitions/InitiativeDeviationResponse"
          }
        },
        "agentJiraUrl": {
          "type": [
            "string",
            "null"
          ],
          "pattern": "^.*$",
          "maxLength": 255,
          "description": "jira url"
        }
      },
      "additionalProperties": false
    },
    "Sort": {
      "type": [
        "object",
        "null"
      ],
      "description": "Sort",
      "properties": {
        "unsorted": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "unsorted"
        },
        "sorted": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "sorted"
        },
        "empty": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "empty"
        }
      },
      "additionalProperties": false
    },
    "PageableSwagger": {
      "type": [
        "object",
        "null"
      ],
      "description": "PageableSwagger",
      "properties": {
        "sort": {
          "$ref": "#/definitions/Sort"
        },
        "pageNumber": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "page number"
        },
        "pageSize": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "page size"
        },
        "offset": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "offset"
        },
        "paged": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "paged"
        },
        "unpaged": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "unpaged"
        }
      },
      "additionalProperties": false
    },
    "PageSwagger": {
      "type": [
        "object",
        "null"
      ],
      "properties": {
        "content": {
          "type": "array",
          "maxItems": 2147483647,
          "items": {
            "$ref": "#/definitions/AiAgentShowcaseResponse"
          }
        },
        "pageable": {
          "$ref": "#/definitions/PageableSwagger"
        },
        "totalElements": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "total elements"
        },
        "totalPages": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "total pages"
        },
        "numberOfElements": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "number of elements"
        },
        "number": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "number"
        },
        "size": {
          "type": [
            "integer",
            "null"
          ],
          "minimum": 0,
          "maximum": 2147483647,
          "description": "size"
        },
        "first": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "first"
        },
        "last": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "last"
        },
        "sort": {
          "$ref": "#/definitions/Sort"
        },
        "empty": {
          "type": [
            "boolean",
            "null"
          ],
          "description": "empty"
        }
      },
      "additionalProperties": false
    }
  }
}
  ```
