# 启动
## 打包启动
```
mvn clean package -DskipTests
cd sky-server
java -jar target/sky-server-1.0-SNAPSHOT.jar
```

## 不打包启动

### 单命令启动
```
mvn -f sky-server/pom.xml spring-boot:run
```
### 先安装依赖，再进入子模块启动
```
mvn clean install -DskipTests
cd sky-server && mvn spring-boot:run
```
