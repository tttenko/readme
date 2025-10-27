```java

/**
 * <b>Given</b> кеш пуст, запрошен один ключ {@code A}.<br>
 * <b>When</b> loader возвращает ровно один элемент для {@code A}.<br>
 * <b>Then</b> элемент кладётся в кеш и метод возвращает singleton-результат.
 *
 * @see org.mockito.ArgumentCaptor
 */
java
Копировать код
/**
 * <b>Given</b> в кеше уже есть запись для {@code A}.<br>
 * <b>When</b> запрошен один ключ {@code A}.<br>
 * <b>Then</b> чистый hit: loader не вызывается, результат берётся из кеша.
 *
 * @see org.mockito.Mockito#never()
 */
java
Копировать код
/**
 * <b>Given</b> кеш пуст, запрошен один ключ {@code B}.<br>
 * <b>When</b> loader возвращает пустой список.<br>
 * <b>Then</b> результат пуст, запись в кеш не создаётся.
 */
java
Копировать код
/**
 * <b>Given</b> кеш пуст, запрошен один ключ {@code N1}.<br>
 * <b>When</b> loader возвращает список, содержащий {@code null}.<br>
 * <b>Then</b> {@code null}-элемент игнорируется: результат пуст, кеш не изменяется.
 */

```
