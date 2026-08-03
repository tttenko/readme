```java
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "$ref": "#/definitions/InitiativeDeviationCountResponse",
  "definitions": {
    "InitiativeDeviationCountResponse": {
      "type": "object",
      "required": [
        "totalInitiativeWithDeviations"
      ],
      "properties": {
        "totalInitiativeWithDeviations": {
          "type": "integer",
          "minimum": 0,
          "maximum": 4503599627370496,
          "description": "Количество инициатив с отклонениями",
          "example": 5
        }
      },
      "additionalProperties": false
    }
  }
}
  ```
