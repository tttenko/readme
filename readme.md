```java
@Override
    public final boolean equals(Object o) {
        if (this == o) return true;
        if (o == null) return false;
        if (Hibernate.getClass(this) != Hibernate.getClass(o)) return false;
        FxRateEntity that = (FxRateEntity) o;
        return id != null && Objects.equals(id, that.id);
    }

    @Override
    public final int hashCode() {
        // важно: стабильный hashCode для transient-объектов (id == null)
        return Hibernate.getClass(this).hashCode();
    }
```
