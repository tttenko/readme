```java

<!-- TMC: генерим api + models -->
                <execution>
                    <id>generate-masterdata-tmc</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-tmc.yaml</inputSpec>
                        <output>${project.build.directory}/generated-sources/swagger/masterdata-tmc</output>

                        <generatorName>spring</generatorName>
                        <library>spring-http-interface</library>

                        <apiPackage>${masterdata.codegen.basepackage}.tmc.api</apiPackage>
                        <modelPackage>${masterdata.codegen.basepackage}.models.md</modelPackage>
                        <invokerPackage>${masterdata.codegen.basepackage}.invoker</invokerPackage>
                        <configPackage>${masterdata.codegen.basepackage}.config</configPackage>

                        <generateApis>true</generateApis>
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

                <!-- BOOK: генерим только api, модели НЕ генерим -->
                <execution>
                    <id>generate-masterdata-book</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/ewm-reference-book.yaml</inputSpec>
                        <output>${project.build.directory}/generated-sources/swagger/masterdata-book</output>

                        <generatorName>spring</generatorName>
                        <library>spring-http-interface</library>

                        <apiPackage>${masterdata.codegen.basepackage}.book.api</apiPackage>
                        <modelPackage>${masterdata.codegen.basepackage}.models.md</modelPackage>
                        <invokerPackage>${masterdata.codegen.basepackage}.invoker</invokerPackage>
                        <configPackage>${masterdata.codegen.basepackage}.config</configPackage>

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
