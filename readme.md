```java
Перевёл вызовы Master-Data с ручного HttpRequestHelper на сгенерированный OpenAPI-клиент (ApiApi) через HttpServiceProxyFactory. Добавил конфиг, который поднимает два бина mdBookApi/mdTmcApi с правильным baseUrl, и общий WebClient-фильтр для логирования и единого маппинга ошибок в Mda*Exception. Вынес вызов МД в MasterDataApiApiClient implements MasterDataClient, а BaseMasterDataRequestService теперь просто делегирует в этот клиент. Тесты на новую инфраструктуру пока не добавлял — PR для ревью именно архитектуры/настроек.
```
