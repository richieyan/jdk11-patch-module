# java 11 patch module

## 背景

java 9 之后移除一些非关键的 internal API， 如 sun.misc.Base64Encoder。

一些第三方 SDK 依赖了 `sun.misc.Base64Encoder` 这种已经被移除的类，如果直接使用 JDK 11 会导致找不到类的异常。

目前，利用 java 9 模块系统提供的 --patch-module 可以将此类模块加回到运行时。

## 实现细节

### SDK 兼容性判断

使用 `jdeps` 可以查询对 Java 9 的兼容情况

```shell
jdeps --multi-release 9  --jdk-internals -R your-sdk.jar

....
JDK 内部 API                               建议的替换
----------                               -----
sun.misc.BASE64Decoder                   Use java.util.Base64 @since 1.8
sun.misc.BASE64Encoder                   Use java.util.Base64 @since 1.8
```

### 模块隶属问题

以 `patch-base64` 项目举例

当直接编译项目时，会提示：

```
sun\misc\BASE64Encoder.java:1: error: package exists in another module: jdk.unsupported
```

从而知道其属于 `java.unsupported` 模块，此模块名用于 maven 编译配置。

### maven 编译插件

配置 `java.unsupported=${project.basedir}/src/main/java`

```xml

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <release>11</release>
                <compilerArgs>
                    <arg>--patch-module</arg>
                    <arg>jdk.unsupported=${project.basedir}/src/main/java</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

使用 mvn 打包
```shell
mvn clean compile jar:jar
```

patch 模块包需要通过 jar 的方式分发，并通过命令行的方式使用。


### 使用 patch 模块包

自己创建简单了示例 maven 项目, 主类如下：

```java
package samples.base64;

import java.nio.charset.StandardCharsets;

public class EncodeUtils {
    public static String encode(byte[] arr) {
        return new BASE64Encoder().encode(arr);
    }

    public static void main(String[] args) {
        System.out.println(encode("test".getBytes(StandardCharsets.UTF_8)));
    }
}
```

#### maven 插件配置如下：

```xml

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <archive>
            <manifest>
                <!--运行jar包时运行的主类，要求类全名-->
                <mainClass>samples.base64.EncodeUtils</mainClass>
                <!-- 是否指定项目classpath下的依赖 -->
                <addClasspath>true</addClasspath>
                <!-- 指定依赖的时候声明前缀 -->
                <classpathPrefix>./</classpathPrefix>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

打包，并将 patch-base64 的 jar 包复制到运行目录，在 Java11 下运行：

```shell
mvn clean compile jar:jar
PATCH_BASE64="--patch-module jdk.unsupported=patch-base64-1.0.jar"
java -jar $PATCH_BASE64 $ADD_OPEN test-base64-1.0-SNAPSHOT.jar
```


