```java

public final class XmlConverters {
  private XmlConverters() {}

  // --- string нормализация (как у тебя) ---
  public static class TrimToNull extends StdConverter<String, String> {
    @Override public String convert(String s) { return trimToNull(s); }
  }

  public static class UpperTrimToNull extends StdConverter<String, String> {
    @Override public String convert(String s) {
      String t = trimToNull(s);
      return t == null ? null : t.toUpperCase(Locale.ROOT);
    }
  }

  public static class NormalizeRubUpper extends StdConverter<String, String> {
    @Override public String convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;
      t = t.toUpperCase(Locale.ROOT);
      return "RUR".equals(t) ? "RUB" : t;
    }
  }

  // --- STRICT: для ROOT (RQUID/RqTm) ---

  /** null/blank -> null (пусть потом проверит сервис или валидатор), но невалидный UUID -> exception */
  public static class ToUuidStrict extends StdConverter<String, UUID> {
    @Override public UUID convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;
      try {
        return UUID.fromString(t);
      } catch (Exception e) {
        throw new IllegalArgumentException("Invalid UUID: " + t, e);
      }
    }
  }

  /** поддержка: "2026-02-09T18:00:06", "2026-02-09T18:00:06Z", "+03:00" и т.п.; невалидное -> exception */
  public static class ToLocalDateTimeLenientStrict extends StdConverter<String, LocalDateTime> {
    @Override public LocalDateTime convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;

      try { return LocalDateTime.parse(t); } catch (Exception ignore) {}
      try { return OffsetDateTime.parse(t).toLocalDateTime(); } catch (Exception ignore) {}
      try { return Instant.parse(t).atOffset(ZoneOffset.UTC).toLocalDateTime(); } catch (Exception ignore) {}

      throw new IllegalArgumentException("Invalid LocalDateTime: " + t);
    }
  }

  // --- LENIENT: для FXRates (useDate/lotSize/value) ---

  /** null/blank -> null; невалидное число -> null (чтобы скипнуть одну запись, а не всё сообщение) */
  public static class ToBigDecimalLenient extends StdConverter<String, BigDecimal> {
    @Override public BigDecimal convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;

      // если вдруг прилетает "12,34"
      t = t.replace(',', '.');

      try {
        return new BigDecimal(t);
      } catch (Exception e) {
        return null;
      }
    }
  }

  /** null/blank -> null; невалидная дата -> null */
  public static class ToLocalDateLenientOrNull extends StdConverter<String, LocalDate> {
    @Override public LocalDate convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;

      try { return LocalDate.parse(t); } catch (Exception ignore) {}
      try { return LocalDateTime.parse(t).toLocalDate(); } catch (Exception ignore) {}
      try { return OffsetDateTime.parse(t).toLocalDate(); } catch (Exception ignore) {}
      try { return Instant.parse(t).atOffset(ZoneOffset.UTC).toLocalDate(); } catch (Exception ignore) {}

      return null;
    }
  }

  // --- helpers ---
  private static String trimToNull(String s) {
    if (s == null) return null;
    String t = s.trim();
    return t.isEmpty() ? null : t;
  }
}
```
