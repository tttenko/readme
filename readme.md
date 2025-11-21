```java
public static <T> void mockPostResponse(
        String url,
        Class<T> clazz,
        HttpRequestHelper httpRequestHelper,
        Predicate<String> bodyPredicate,
        T whenTrue,
        T whenFalse
) {
    when(httpRequestHelper.sendPostRequest(eq(url), anyString(), eq(clazz)))
            .thenAnswer(invocation -> {
                String body = invocation.getArgument(1, String.class);
                return bodyPredicate.test(body) ? whenTrue : whenFalse;
            });
}
```
