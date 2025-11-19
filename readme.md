```java

@ExtendWith(MockitoExtension.class)
class ServiceCallExecutorTest {

    @InjectMocks
    private ServiceCallExecutor executor;

    @Mock
    private Supplier<ResultObj<List<String>>> supplier;

    @Mock
    private ResultObj<List<String>> resultObj;

    @Test
    void executeOrThrow_shouldReturnResult_whenCountGreaterThanZero() {
        // given
        when(supplier.get()).thenReturn(resultObj);
        when(resultObj.getCount()).thenReturn(1);

        // when
        ResultObj<List<String>> actual = executor.executeOrThrow(supplier);

        // then
        assertSame(resultObj, actual);
        verify(supplier).get();
        verify(resultObj).getCount();
        verifyNoMoreInteractions(supplier, resultObj);
    }

    @Test
    void executeOrThrow_shouldThrowException_whenCountIsZero() {
        // given
        when(supplier.get()).thenReturn(resultObj);
        when(resultObj.getCount()).thenReturn(0);

        // when / then
        assertThrows(
                MdaDataNotFoundException.class,
                () -> executor.executeOrThrow(supplier)
        );

        verify(supplier).get();
        verify(resultObj).getCount();
        verifyNoMoreInteractions(supplier, resultObj);
    }
}
```
