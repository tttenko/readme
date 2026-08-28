```java

{
  "id": "ee5e8977-8120-40cc-8a4f-d77604320fd6",
  "name": "20",
  "unit": "percent",
  "direction": "more_is_better",
  "agentType": "autonomous",
  "isActive": true,
  "description": "1",
  "frequency": "constant",
  "metricValue": -312,
  "targetValue": null,
  "planExecution": null,
  "periods": [
    {
      "index": 1,
      "period": "2026-06-01T00:00:00Z",
      "value": -280
    }
  ]
}

Если сейчас август, то здесь:

metricValue = значение за июль
periods[0]  = значение за июнь

Если за июнь значения нет, то:

"periods": []
```
