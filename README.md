```java
@ExtendWith(MockitoExtension.class)
class GlobalExceptionHandlerTest {

    @Mock
    private MessageSource messageSource;

    @Mock
    private WebRequest webRequest;

    @Mock
    private ConstraintViolation<?> constraintViolation;

    private GlobalExceptionHandler globalExceptionHandler;

    @BeforeEach
    void setUp() {
        globalExceptionHandler = new GlobalExceptionHandler();
        globalExceptionHandler.setMessageSource(messageSource);
    }

    @Test
    void givenConstraintViolationException_whenHandleBadRequestException_thenReturnBadRequestResponse() {
        // given
        String validationMessage = "Параметр contractUuid не должен быть null";
        String localizedMessage = "Некорректный запрос: " + validationMessage;

        when(constraintViolation.getMessage()).thenReturn(validationMessage);
        when(messageSource.getMessage(
                eq("error.incorrectRequest"),
                aryEq(new Object[]{validationMessage}),
                any(Locale.class)
        )).thenReturn(localizedMessage);

        ConstraintViolationException exception =
                new ConstraintViolationException(Set.of(constraintViolation));

        // when
        ResponseEntity<Object> response =
                globalExceptionHandler.handleBadRequestException(exception, webRequest);

        // then
        assertThat(response).isNotNull();
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody()).isInstanceOf(String.class);
        assertThat(response.getBody()).isEqualTo(localizedMessage);

        verify(messageSource).getMessage(
                eq("error.incorrectRequest"),
                aryEq(new Object[]{validationMessage}),
                any(Locale.class)
        );
        verifyNoMoreInteractions(messageSource);
    }

    @Test
    void givenEntityNotFoundException_whenHandleEntityNotFoundException_thenReturnNotFoundResponse() {
        // given
        String exceptionMessage = "Запись СТС не найдена по uuid: 123e4567-e89b-12d3-a456-426614174000";
        EntityNotFoundException exception = new EntityNotFoundException(exceptionMessage);

        // when
        ResponseEntity<Object> response =
                globalExceptionHandler.handleEntityNotFoundException(exception, webRequest);

```
