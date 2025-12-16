```java

 <!-- 1) MODELS from BOOK -> общий output -->
                <execution>
                    <id>generate-masterdata-models-book</id>
                    <phase>generate-sources</phase>
                    <goals><goal>generate</goal></goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
                        <output>${openapi.out.dir}/masterdata-models</output>

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

                <!-- 2) MODELS from TMC -> ДОПИСЫВАЕМ в тот же output -->
                <execution>
                    <id>generate-masterdata-models-tmc</id>
                    <phase>generate-sources</phase>
                    <goals><goal>generate</goal></goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-tmc.yaml</inputSpec>
                        <output>${openapi.out.dir}/masterdata-models</output>

                        <generatorName>spring</generatorName>
                        <library>spring-http-interface</library>

                        <modelPackage>${openapi-generator.base-package}.models.md</modelPackage>

                        <generateApis>false</generateApis>
                        <generateModels>true</generateModels>
                        <generateSupportingFiles>false</generateSupportingFiles>

                        <!-- важно: не удалять результат book-моделей -->
                        <cleanupOutput>false</cleanupOutput>
                        <!-- чтобы одинаковые модели не перетирались -->
                        <skipOverwrite>true</skipOverwrite>

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

                <!-- 3) TMC API only -->
                <execution>
                    <id>generate-masterdata-tmc-api</id>
                    <phase>generate-sources</phase>
                    <goals><goal>generate</goal></goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-tmc.yaml</inputSpec>
                        <output>${openapi.out.dir}/masterdata-tmc</output>

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

                <!-- 4) BOOK API only -->
                <execution>
                    <id>generate-masterdata-book-api</id>
                    <phase>generate-sources</phase>
                    <goals><goal>generate</goal></goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
                        <output>${openapi.out.dir}/masterdata-book</output>

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
