---
icon: simple/apachemaven
---

maven ~/.m2/中常用的几个配置文件

!!! example

    === "settings.xml"
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <settings xmlns="http://maven.apache.org/SETTINGS/1.1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd">

            <servers>
                <server>
                    <id>data-management-releases</id>
                    <username>junzhzhoupartner</username>
                    <password>{password}</password>
                </server>
                <server>
                    <id>data-management-snapshots</id>
                    <username>junzhzhoupartner</username>
                    <password>{password}</password>
                </server>
                <server>
                    <id>smardci-artifactory</id>
                    <username>junzhzhoupartner</username>
                    <password>{password}</password>
                </server>
                <server>
                    <id>refdata-protobufs-releases</id>
                    <username>junzhzhoupartner</username>
                    <password>{password}</password>
                </server>
                <server>
                    <id>swh-external-here-platform</id>
                    <username>junzhzhoupartner</username>
                    <password>{password}</password>
                </server>
            </servers>

            <proxies>
                <proxy>
                    <id>clash</id>
                    <active>true</active>
                    <protocol>http</protocol>
                    <host>127.0.0.1</host>
                    <port>7890</port>
                    <nonProxyHosts></nonProxyHosts>
                </proxy>
            </proxies>

        </settings>
        ```
    === "toolchains.xml"
        ```xml
        <?xml version="1.0" encoding="UTF8"?>
        <toolchains>
            <toolchain>
                <type>jdk</type>
                <provides>
                    <id>jdk8</id>
                    <version>8</version>
                    <vendor>openjdk</vendor>
                </provides>
                <configuration>
                    <!-- On default LiDec: /usr/lib/jvm/java-8-openjdk-amd64 -->
                    <jdkHome>/ADD/PATH/TO/JDK8</jdkHome>
                </configuration>
            </toolchain>
            <toolchain>
                <type>jdk</type>
                <provides>
                    <id>jdk17</id>
                    <version>17</version>
                    <vendor>openjdk</vendor>
                </provides>
                <configuration>
                    <!-- On default LiDec: /usr/lib/jvm/java-1.17.0-openjdk-amd64 -->
                    <jdkHome>/ADD/PATH/TO/JDK17</jdkHome>
                </configuration>
            </toolchain>
        </toolchains>
        ```
    === "jvm.config"
        ```text
        -Xmx8G
        -Xss128M
        -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled  -Duser.timezone=GMT -Dorg.slf4j.simpleLogger.showDateTime=true -Dorg.slf4j.simpleLogger.dateTimeFormat=yyyy-MM-dd'"'T'"'HH:mm:ss,SSSZ
        ```

### 常用的maven命令或参数
在monorepo项目中，当maven项目结构比较复杂以下命令或参数比较有用

#### 跳过所有测试的参数
```
-DskipTests -Dunit.skipTests -Dintegration.skipTests -Dscoverage.skip -Djacoco.skip
```

#### Building specific sub-module only
```bash
mvn clean package -am -pl gt-generation/gt-generation-spark
```

#### Run single test in module
```bash
mvn test -pl :labeler -Dsuites='com.sample.test.DayTimeLabelerTest'
```

### 增加公司Copyright头

maven license plugin is configured to add
[copyright-header.txt](../../templates/copyright-header.txt) to src files.

- Use `mvn license:check` to check if files compliance with copyright header
- Use `mvn license:format` to format files based on predefined copyright header