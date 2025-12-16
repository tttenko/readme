```java

 <!-- TMC: API + MODELS (модели оставляем в старом пакете, чтобы бизнес-код не менять) -->
    <execution>
      <id>generate-masterdata-tmc</id>
      <phase>generate-sources</phase>
      <goals>
        <goal>generate</goal>
      </goals>
      <configuration>
        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-tmc.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/swagger/masterdata-tmc</output>

        <generatorName>spring</generatorName>
        <library>spring-http-interface</library>

        <apiPackage>${openapi-generator.base-package}.tmc.api</apiPackage>
        <!-- ВАЖНО: этот пакет остаётся как сейчас -->
        <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

        <generateApis>true</generateApis>
        <generateModels>true</generateModels>
        <generateSupportingFiles>false</generateSupportingFiles>

        <addCompileSourceRoot>true</addCompileSourceRoot>

        <configOptions>
          <useJakartaEe>true</useJakartaEe>
          <reactive>false</reactive>
          <interfaceOnly>true</interfaceOnly>
          <useRuntimeException>true</useRuntimeException>
          <openApiNullable>true</openApiNullable>
          <dateLibrary>java8-localdatetime</dateLibrary>
          <hideGenerationTimestamp>true</hideGenerationTimestamp>
          <hideGeneratorVersion>true</hideGeneratorVersion>
          <sourceFolder>src/main/java</sourceFolder>
        </configOptions>
      </configuration>
    </execution>

    <!-- BOOK: API + MODELS (модели отдельно, чтобы не конфликтовали и “лежали рядом”) -->
    <execution>
      <id>generate-masterdata-book</id>
      <phase>generate-sources</phase>
      <goals>
        <goal>generate</goal>
      </goals>
      <configuration>
        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/swagger/masterdata-book</output>

        <generatorName>spring</generatorName>
        <library>spring-http-interface</library>

        <apiPackage>${openapi-generator.base-package}.book.api</apiPackage>
        <!-- BOOK-модели в отдельном пакете -->
        <modelPackage>${openapi-generator.base-package}.book.models.md</modelPackage>

        <generateApis>true</generateApis>
        <generateModels>true</generateModels>
        <generateSupportingFiles>false</generateSupportingFiles>

        <addCompileSourceRoot>true</addCompileSourceRoot>

        <configOptions>
          <useJakartaEe>true</useJakartaEe>
          <reactive>false</reactive>
          <interfaceOnly>true</interfaceOnly>
          <useRuntimeException>true</useRuntimeException>
          <openApiNullable>true</openApiNullable>
          <dateLibrary>java8-localdatetime</dateLibrary>
          <hideGenerationTimestamp>true</hideGenerationTimestamp>
          <hideGeneratorVersion>true</hideGeneratorVersion>
          <sourceFolder>src/main/java</sourceFolder>
        </configOptions>
      </configuration>
    </execution>
```
