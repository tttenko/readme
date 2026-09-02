```java
План принимаю после следующих финальных корректировок:

Исправь @GetMapping: не использовать MediaType.APPLICATION_OCTET_STREAM_VALUE. API-014 должен возвращать application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.
Для FR-029 сравнивай этапы не с raw agent.agentStatus.ordering, а с текущим витринным статусом инициативы. Учти существующую нормализацию research → analysis, release → targetSolution; analysis/development/pilot/targetSolution остаются без изменения. Не дублируй нормализацию, если в backend уже существует подходящий helper/mapping.
Email объединять строго через ";" без добавления пробела: joinToString(";").
Не задавай production URL как default непосредственно в PultProperties.kt. URL должен приходить из конфигурации/application.yaml/environment. Service не должен содержать production hardcode.
Добавь тесты FR-029 для служебных текущих статусов research и release, подтверждающие их нормализацию в analysis и targetSolution.
Для Excel hyperlink добавь проверку именно свойства hyperlink ячейки (cell.hyperlink.address), а не только строкового URL в DTO.

Остальной план сохраняй. После этих корректировок реализацию можно начинать.
```
