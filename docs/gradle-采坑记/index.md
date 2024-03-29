# Gradle 采坑记


最近开始尝试使用 Gradle 构建项目，踩了一些坑，简略记录以备忘。
<!--more-->
Gradle 项目可以使用 Gradle Wrapper (简称 Wrapper)，也可以不使用。Wrapper 的主要目的是将项目构建依赖的 Gradle 版本配置化，作为构建脚本的一部分。Wrapper 只是一个脚本，它调用指定版本的 Gradle；如果指定版本的 Gradle 不存在，就下载并自动安装，避免开发者手动安装。官方推荐使用 Wrapper 执行任何 Gradle build。可是墙内下载 Gradle 非常慢，一开始还不知道为啥必须下载，如何加速，这让人很懊恼。如何使用 Wrapper，请看 [The Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html)。

解决办法有两个，但都需要先手工把对应版本下载到本地。

**办法一：**

1. 部署 HTTP Server (如 Nginx)，把下载的文件放入站点目录。
2. 修改 {Gradle Project}/gradle/wrapper/gradle-wrapper.properties 文件，将 distributionUrl 设置为本地或内网的可用下载地址。

   ```properties
   distributionBase=GRADLE_USER_HOME
   distributionPath=wrapper/dists
   #distributionUrl=https\://services.gradle.org/distributions/gradle-6.6.1-bin.zip
   distributionUrl=http\://localhost:8080/gradle-6.6.1-bin.zip
   zipStoreBase=GRADLE_USER_HOME
   zipStorePath=wrapper/dists
   ```

   注意，对于 properties 文件，name/value 的分隔符是 '=' 或 ':'，所以如果 value 中包含这两个字符，记得用 \ 进行转义。

**办法二：**

1. 把下载文件放到本地某个目录，例如：/Users/{user}/Cellar/gradle-6.6.1-bin.zip

2. 修改 {Gradle Project}/gradle/wrapper/gradle-wrapper.properties 文件，将 distributionUrl 设置为本地文件路径：

   ```properties
   distributionBase=GRADLE_USER_HOME
   distributionPath=wrapper/dists
   #distributionUrl=https\://services.gradle.org/distributions/gradle-6.6.1-bin.zip
   distributionUrl=file\:///Users/xxx/Cellar/gradle-6.6.1-bin.zip
   zipStoreBase=GRADLE_USER_HOME
   zipStorePath=wrapper/dists
   ```

   最后，在项目根目录运行命令 `./gradlew build` 检查效果如何。

