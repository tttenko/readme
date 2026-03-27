```java
@ExtendWith(MockitoExtension.class)
class StsTrackerHistoryListenerTest {

    @Mock
    private StsTrackerHistoryService stsTrackerHistoryService;

    @Mock
    private StsCreatedTrackerHistoryEvent event;

    @InjectMocks
    private StsTrackerHistoryListener stsTrackerHistoryListener;

    @Test
    void handle_shouldDelegateEventToHistoryService() {
        stsTrackerHistoryListener.handle(event);

        verify(stsTrackerHistoryService).sendCreatedStatus(event);
        verifyNoMoreInteractions(stsTrackerHistoryService);
    }
}
```
