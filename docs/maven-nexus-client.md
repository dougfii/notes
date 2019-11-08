# Maven 私有仓库本地配置
1. 配置 .m2 目录下 settings.xml
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
       <servers>
         <!-- for nexus -->
         <server>
           <id>xxx-repositories</id>
           <username>your-username</username>
           <password>your-password</password>
         </server>
         <!-- for dockerfile-maven-plugin -->
         <server>
            <id>registry.cn-hangzhou.aliyuncs.com</id>
           <username>your-registry-username</username>
           <password>your-registry-password</password>
         </server>
       </servers>
    </settings>
    ```
2. 工程 pom.xml 中加入
    ```
    <distributionManagement>
        <repository>
            <id>xxx-repositories</id>
            <name>xxx-releases</name>
            <url>http://private.nexus.domain/repository/xxx-releases/</url>
        </repository>
        <snapshotRepository>
            <id>xxx-repositories</id>
            <name>xxx-snapshots</name>
            <url>http://private.nexus.domain/repository/xxx-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
    ```