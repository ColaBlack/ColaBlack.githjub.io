参考：

1.[1Panel 官方文档](https://1panel.cn/docs/installation/online_installation/)



本文介绍一下如何利用1panel部署一个简单的前后端分离项目。

### 1,拥有一个Liunx服务器

第一步是购买一个Linux服务器，可以买一台线下真实的机器+公网IP或买一个阿里云、腾讯云、京东云、华为云服务器。

### 2.安装1panel

参考1panel官方文档，安装1panel。

在这里以Ubuntu系统为例，只需运行

```bash
curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && sudo bash quick_start.sh
```

即可安装，安装完成后终端上有写1panel的URL和账号名密码。

### 3.安装运行环境

在虚拟机中安装java等运行环境。

然后，可以在1panel中安装项目需要的中间件，如MySQL、minio、redis等。

### 4.打包后端项目

先介绍一下如何打包maven项目。

需要注意的是，你可能会发现在idea的maven菜单里，已经有一个package选项了，然而默认情况下这样打的包是不带项目依赖的。所以这样的jar包不能独立运行。

但是如果你用的是spring boot项目，则pom.xml中可能已经安装了插件spring-boot-maven-plugin

```xml
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
```

如果pom.xml有这样的代码则说明maven有插件spring-boot-maven-plugin。

这时直接在idea的maven菜单运行package选项就得到带依赖的jar包了，可以直接java -jar运行。

以部署笔者的teaai项目为例，打包后你会在target文件夹下看到teaai-backend-0.0.1-SNAPSHOT.jar，这就是打包后的jar文件，直接运行即可启动后端服务。

对于非spring boot项目又该如何打包呢？

需要安装maven-assembly-plugin插件，方法如下：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>3.3.0</version><!-- 插件版本号 -->
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifest>
                        <mainClass>edu.zafu.teaai.MainApplication</mainClass>
                        <!-- 替换为你的主类的完整类名 -->
                    </manifest>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

然后再运行package即可。

对于gradle项目，没有什么太大的区别。

这里只介绍一般的gradle项目如何打带依赖的jar包。

gradle依赖打包需要安装shadow插件，在build.gradle.kts中的plugins，增加这样的代码：

```kotlin
plugins {
    id("com.gradleup.shadow") version "8.3.3"
}
```

然后增加shadow任务，同样还是在build.gradle.kts中增加一下代码：

```kotlin
tasks {
    shadowJar {
        archiveBaseName.set("nocrud")
        archiveVersion.set("")
        manifest {
            attributes["Main-Class"] = "cn.cola.nocrud.MainKt"
        }
    }
}
```

这段代码以笔者的nocrud项目为例，在上面的代码中nocrud是你项目的名称，`attributes["Main-Class"] = "cn.cola.nocrud.MainKt"`则规定了主类。

需要注意的是，如果你的主类是一个kotlin代码，则需要再原本的类名后面加上Kt，这是因为kotlin是一个jvm语言，编译后你会发现kotlin编译器会在所有的kotlin类名称后面加上Kt。

完成了这些，刷新一下项目，你会发现idea的gradle菜单中新增了一个shadow任务，双击shadowJar命令执行即可打包。

打包完的jar包在build/libs目录内。

### 5.启动后端项目

在1panel-主机-文件中上传打包后的jar包。

然后到网站-运行环境-java中运行后端项目。

当然如果你的后端是go语言或php或nodejs那么就去对应的页面。

本文以java项目为例，选择创建运行环境，设置名称和java sdk版本，将运行目录设置为jar包的上传目录。

在启动命令一栏输入完整的启动命令，如

```bash
java -jar ./teaai-backend-0.0.1-SNAPSHOT.jar
```

然后注意，1panel和宝塔面板有所不同，1panel的后端项目也是基于容器化部署的，需要填写应用在容器内的访问端口和容器外的端口。

然后打开端口外部访问并设置容器名称。

点击确定即可。

最后提醒一下，因为1panel是容器化部署，而在容器内localhost指向的是容器内部的地址，如果想访问容器网外的本机的其他容器请使用本机的真实内网IP。

### 6.部署其他中间件

这里需要根据中间件的不同进行部署，如关系型数据库需要建表等。

### 7.打包前端项目

这里以打包teaai项目为例，teaai是一个vue项目。

用WebStrom打开teaai前端项目，一般而言当你利用vue-cli创建项目时，在package.json中有这样的命令。

```json
    "build": "run-p type-check \"build-only {@}\" --",
```

你可以直接点击WebStorm左边的运行按钮运行，也可以在终端中输入

```bash
npm run build
```

打包项目。

当然，如果你用pnpm或yarn等运行这个命令也一样。

打包后你将在dist目录下看到打包后的结果。

### 8.启动前端项目

进入1panel-网站-运行环境-php创建一个php运行环境，拓展模版选择“默认”即可。

然后去你的DNS服务商配置域名解析（这个因服务商而不同，在此无法演示）。

接下来进入1panel面板-网站-网站，如果你没有安装OpenResty（你可以理解为nginx增强版）则1panel会提示你安装。

在网站页面选择创建网站-运行环境-选择刚刚创建的php运行环境

（如果你的网站是静态网站也可以不创建php运行环境而直接选静态网站）

输入网站的域名和需要访问的端口号,点击确定。

你会看到1panel页面的表格中多了一条记录，点击其中的网站目录下的文件夹图标，进入文件界面上传前端打包的dist目录的内容。

接着。如果要设置代理和https服务的话，回到刚刚的1panel-网站-网站页面，点击网站记录右边的配置，在这里可以对网站进行限流、反向代理、配置HTTPS服务等操作。

如果开启了https服务，记得在防火墙打开443端口！