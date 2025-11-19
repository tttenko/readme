```java

GetItemsSearchResponse searchResponse =
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForCounterparty(),
                        criteria
                );

        try {
            byte[] bytes = objectMapper.writeValueAsBytes(searchResponse);
            log.info("GetItemsSearchResponse size ~= {} bytes ({} KB)",
                    bytes.length, bytes.length / 1024);
        } catch (JsonProcessingException e) {
            log.warn("Cannot calculate response size", e);
        }
```
