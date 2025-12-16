```java

<!-- 1) MODELS: генерим ОДИН раз (лучше из book, т.к. он “толще”) -->
    <execution>
      <id>generate-masterdata-models</id>
      <goals><goal>generate</goal></goals>
      <configuration>
        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/swagger/masterdata-models</output>

        <generatorName>spring</generatorName>
        <library>spring-http-interface</library>

        <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

        <generateApis>false</generateApis>
        <generateModels>true</generateModels>
        <generateSupportingFiles>false</generateSupportingFiles>

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

    <!-- 2) TMC: генерим только API, модели НЕ генерим -->
    <execution>
      <id>generate-masterdata-tmc</id>
      <goals><goal>generate</goal></goals>
      <configuration>
        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-tmc.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/swagger/masterdata-tmc</output>

        <generatorName>spring</generatorName>
        <library>spring-http-interface</library>

        <apiPackage>${openapi-generator.base-package}.tmc.api</apiPackage>
        <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

        <generateApis>true</generateApis>
        <generateModels>false</generateModels>
        <generateSupportingFiles>false</generateSupportingFiles>

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

    <!-- 3) BOOK: генерим только API, модели НЕ генерим -->
    <execution>
      <id>generate-masterdata-book</id>
      <goals><goal>generate</goal></goals>
      <configuration>
        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/swagger/masterdata-book</output>

        <generatorName>spring</generatorName>
        <library>spring-http-interface</library>

        <apiPackage>${openapi-generator.base-package}.book.api</apiPackage>
        <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

        <generateApis>true</generateApis>
        <generateModels>false</generateModels>
        <generateSupportingFiles>false</generateSupportingFiles>

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
