---
title: JDK 导入 (CA) SSL 认证-纯净版
date: 2024-11-15 11:32:29
categories:
  - tools
tags:
  - java
  - 杂项
---
JDK 提供 keytool 是密钥和证书管理工具，可以生成证书、导入证书、生成密钥。可以给通过 java 运行程序 https 安全认证，当然不包含tomcat、jetty 容器，他们有自己证书配置。默认 keytool 密码库是 `$JAVA_HOME/jre/lib/security/cacerts`，如果自身有需要可以通过 `keytool -keystore filepath` 其中 filepath 可以是绝对路径，也可以仅是文件名，区别在于是在本文件夹下的密码库还是指定文件夹的密码库，**这点很重要**。

我是在 sonar-maven-plugin 使用时，因 sonar 是 https 注意到 JDK SSL 认证。期间碰到了一些问题，问题都是踩过的坑。

- ` sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target`

    - <font color='blue'>没有指定密钥库或密钥库中没有有效的密钥</font>

- `java.security.UnrecoverableKeyException: Password verification failed`

    - <font color='blue'>指定的密钥库对应的密码不正确，当然对应的路径下可能不存在密钥库</font>

- `java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty`

    - <font color='blue'>对应的密码库路径不正确</font>


### JDK 导入 CA 认证

```shell
keytool -importcert -keystore <keystore-file-path> -alias <caAlias> -file <ca-file-path>
```

- keystore，指定密钥库路径。如果仅文件名，则默认在运行命令的目录创建密钥库。

- alias，只是用来方便标记认证别名。

- file，指向 ca 证书


运行命令时，需要设置密码库的使用密码，**这个千万不能忘记**。运行命令后，会在 `keystore-file-path` 生成相应的密码库文件。可以使用 `keytool -list -v -keystore <keystore-file-path>` 查看导入证书明细。

### mvn sonar 运行变量指定

`mvn clean compile sonar:sonar -Djavax.net.ssl.trustStore=<keystore-file-path> -Djavax.net.ssl.trustStorePassword=<keystore-password>`

- `javax.net.ssl.trustStore`，指向密码库的绝对路径

- `javax.net.ssl.trustStorePassword`，指向密码库的使用密码