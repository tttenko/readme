```java

CountDownLatch startBarrier = new CountDownLatch(1);   // чтобы оба лоадера стартовали одновременно
  CountDownLatch workBarrier  = new CountDownLatch(1);   // вместо Thread.sleep: «имитация работы»

  AtomicInteger loaderCalls = new AtomicInteger(0);

  Function<List<String>, List<Tb>> loader = keys -> {
    loaderCalls.incrementAndGet();
    try {
      // дождаться разрешения на старт от теста
      startBarrier.await(1, TimeUnit.SECONDS);
      // «поработать», ожидая сигнала от теста (не sleep)
      workBarrier.await(250, TimeUnit.MILLISECONDS);
    } catch (InterruptedException ignored) { }
    List<Tb> out = new ArrayList<>();
    for (String k : keys) {
      out.add(new Tb(k, "Name " + k));
    }
    return out;
  };

  ExecutorService pool = Executors.newFixedThreadPool(2);
  try {
    Callable<List<Tb>> task1 = () -> batch.fetchBatch(
        CACHE_NAME, Arrays.asList("A", "B", "C"), loader, Tb::getCode, Tb.class);
    Callable<List<Tb>> task2 = () -> batch.fetchBatch(
        CACHE_NAME, Arrays.asList("B", "C", "D"), loader, Tb::getCode, Tb.class);

    Future<List<Tb>> f1 = pool.submit(task1);
    Future<List<Tb>> f2 = pool.submit(task2);

    // одновременно запускаем лоадеры
    startBarrier.countDown();
    // даём им «поработать» (ожидание на workBarrier), после чего разрешаем завершиться
    workBarrier.countDown();

    List<Tb> r1 = f1.get(2, TimeUnit.SECONDS);
    List<Tb> r2 = f2.get(2, TimeUnit.SECONDS);

    assertEquals(3, r1.size());
    assertEquals(3, r2.size());

    Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
    assertEquals(Set.of("A", "B", "C", "D"), keys);

    // допускаем максимум 2 batch-вызова (по числу конкурентных запросов), точно не по одному на ключ
    assertTrue(loaderCalls.get() <= 2, "Loader must not be called per-key (no N+1)");
  } finally {
    pool.shutdownNow();
  }


```
