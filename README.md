```java
@Configuration
@EnableJpaAuditing(
        dateTimeProviderRef = "zonedDateTimeProvider",
        auditorAwareRef = "auditorAware"
)
public class JpaConfig {

    private static final String USER_PROFILE_UUID = "USER_PROFILE_UUID";

    @Bean
    public DateTimeProvider zonedDateTimeProvider() {
        return () -> Optional.of(ZonedDateTime.now());
    }

    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> {
            AuthorizedUser authorizedUser = SecurityUtils.getCurrentUser();
            String userProfileUuid = extractUserProfileUuid(authorizedUser);

            if (userProfileUuid == null || userProfileUuid.isBlank()) {
                throw new AuthenticationServiceException(
                        "USER_PROFILE_UUID отсутствует в metadata.tech текущего пользователя"
                );
            }

            return Optional.of(userProfileUuid);
        };
    }

    @SuppressWarnings("unchecked")
    private String extractUserProfileUuid(AuthorizedUser authorizedUser) {
        if (authorizedUser == null || authorizedUser.getMetadata() == null) {
            return null;
        }

        Object tech = authorizedUser.getMetadata().getTech();
        if (!(tech instanceof Map<?, ?> techMap)) {
            return null;
        }

        Object value = techMap.get(USER_PROFILE_UUID);
        return value != null ? value.toString() : null;
    }
}
```
