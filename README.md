```java

Привет.

После пуша веток master и develop в репозиторий control-vehicle-app pipeline падает на этапе Sonar.

Ошибка в логе такая:

Failed to execute goal org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar ... You're not authorized to run analysis. Please contact the project administrator.

Похоже, что у Jenkins/service account или токена, под которым запускается анализ, нет прав на выполнение Sonar analysis для этого проекта, либо не настроен сам проект в Sonar.


```
