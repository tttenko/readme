```java
<dependencies>
    <!-- Spring: http, web, format, validation.annotation -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>

    <!-- Jakarta Validation / Annotation (так как в генераторе useJakartaEe=true) -->
    <dependency>
        <groupId>jakarta.validation</groupId>
        <artifactId>jakarta.validation-api</artifactId>
    </dependency>
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
    </dependency>

    <!-- Jackson аннотации: @JsonProperty, @JsonInclude и т.п. -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-annotations</artifactId>
    </dependency>

    <!-- Swagger / OpenAPI аннотации (@Operation, @Parameter, …) -->
    <dependency>
        <groupId>io.swagger.core.v3</groupId>
        <artifactId>swagger-annotations</artifactId>
    </dependency>
</dependencies>

```
