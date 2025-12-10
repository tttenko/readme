```java
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>${openapi-generator-maven-plugin.version}</version>

    <executions>
        <execution>
            <id>generate-masterdata-client</id>
            <goals>
                <goal>generate</goal>
            </goals>

            <configuration>
                <!-- метаданные артефакта, как в соседнем модуле -->
                <groupId>${openapi-generator.groupId}</groupId>
                <artifactId>${openapi-generator.artifactId}</artifactId>
                <artifactVersion>${project.version}</artifactVersion>

                <!-- твой YAML -->
                <inputSpec>
                    ${project.basedir}/src/main/resources/ewm-reference-tmc.yaml
                </inputSpec>

                <!-- куда класть сгенерённый код -->
                <output>${project.build.directory}/generated-sources/swagger/masterdata</output>

                <!-- тип генератора -->
                <generatorName>spring</generatorName>
                <library>spring-http-interface</library>

                <!-- пакеты такие же, как были, чтобы не ломать импорт в master-data -->
                <apiPackage>${openapi-generator.base-package}.api</apiPackage>
                <invokerPackage>${openapi-generator.base-package}.invoker</invokerPackage>
                <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

                <!-- что генерим -->
                <generateApis>true</generateApis>
                <generateModels>true</generateModels>
                <generateApiTests>false</generateApiTests>
                <generateModelTests>false</generateModelTests>
                <generateSupportingFiles>false</generateSupportingFiles>
                <configHelp>false</configHelp>

                <configOptions>
                    <useJakartaEe>true</useJakartaEe>
                    <reactive>false</reactive>

                    <!-- только интерфейсы клиента, без контроллеров -->
                    <interfaceOnly>true</interfaceOnly>

                    <useRuntimeException>true</useRuntimeException>

                    <!-- тот же basePackage, что и выше -->
                    <basePackage>${openapi-generator.base-package}</basePackage>
                    <configPackage>${openapi-generator.base-package}.config</configPackage>

                    <!-- nullable через jackson-databind-nullable -->
                    <openApiNullable>true</openApiNullable>

                    <!-- даты как в твоём варианте -->
                    <dateLibrary>java8-localdatetime</dateLibrary>

                    <hideGenerationTimestamp>true</hideGenerationTimestamp>
                    <hideGeneratorVersion>true</hideGeneratorVersion>

                    <!-- важен для пути: .../src/main/java -->
                    <sourceFolder>src/main/java</sourceFolder>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>

```
