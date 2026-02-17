```java/**
class FxRateEntityEqualsHashCodeTest {

    @Test
    void equals_whenSameInstance_thenTrue() {
        FxRateEntity e = new FxRateEntity();
        assertThat(e.equals(e)).isTrue();
    }

    @Test
    void equals_whenOtherIsNull_thenFalse() {
        FxRateEntity e = new FxRateEntity();
        assertThat(e.equals(null)).isFalse();
    }

    @Test
    void equals_whenOtherDifferentType_thenFalse() {
        FxRateEntity e = new FxRateEntity();
        assertThat(e.equals("x")).isFalse();
    }

    @Test
    void equals_whenIdNull_thenFalseEvenIfOtherAlsoIdNull() {
        FxRateEntity a = new FxRateEntity();
        FxRateEntity b = new FxRateEntity();

        // оба id = null
        assertThat(a.equals(b)).isFalse();
        assertThat(b.equals(a)).isFalse();
    }

    @Test
    void equals_whenSameNonNullId_thenTrue_andHashCodeConsistent() {
        FxRateEntity a = new FxRateEntity();
        FxRateEntity b = new FxRateEntity();

        UUID id = UUID.randomUUID();
        setId(a, id);
        setId(b, id);

        assertThat(a.equals(b)).isTrue();
        assertThat(b.equals(a)).isTrue();
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void equals_whenDifferentNonNullId_thenFalse() {
        FxRateEntity a = new FxRateEntity();
        FxRateEntity b = new FxRateEntity();

        setId(a, UUID.randomUUID());
        setId(b, UUID.randomUUID());

        assertThat(a.equals(b)).isFalse();
        assertThat(b.equals(a)).isFalse();
    }

    // --- helper: выставляем private UUID id ---

    private static void setId(FxRateEntity e, UUID id) {
        try {
            // если у вас есть public setId(UUID) — можно заменить на e.setId(id);
            Field f = FxRateEntity.class.getDeclaredField("id");
            f.setAccessible(true);
            f.set(e, id);
        } catch (Exception ex) {
            throw new RuntimeException("Cannot set FxRateEntity.id for test", ex);
        }
    }
}

```
