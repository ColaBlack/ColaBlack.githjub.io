### 参考资料

[FreeMarker官方文档（英文）]([Apache FreeMarker Manual](https://freemarker.apache.org/docs/index.html))

[FreeMarker 中文官方参考手册](http://freemarker.foofun.cn/toc.html)

[Picocli官方文档（英文）](https://picocli.info/#_example_application)

[picocli-中文博客](https://blog.csdn.net/it_freshman/article/details/125458116)

### 1.安装依赖

```kotlin
    // https://mvnrepository.com/artifact/org.freemarker/freemarker
implementation("org.freemarker:freemarker:2.3.33")
// https://mvnrepository.com/artifact/info.picocli/picocli
implementation("info.picocli:picocli:4.7.6")
```

### 2.编写Freemarker demo

参考官方文档，编写一个简单Freemarker的demo，生成一个Java文件。

```kotlin
import freemarker.template.Configuration
import freemarker.template.TemplateExceptionHandler
import java.io.File
import java.io.OutputStreamWriter


fun main() {
    // 创建配置对象
    val config = Configuration(Configuration.VERSION_2_3_33)
    // 设置模板文件存放的目录
    config.setDirectoryForTemplateLoading(File("src/main/resources/templates"))
    // 设置默认的编码格式
    config.defaultEncoding = "UTF-8"
    // 设置异常处理器
    config.templateExceptionHandler = TemplateExceptionHandler.RETHROW_HANDLER
    /*
    利用HashMap创建数据模型
    {
    "user": "Big Joe",
    "latestProduct": {
        "url": "https://github.com/ColaBlack",
        "name": "No CRUD"
    }
     */
    val hashMap = HashMap<String, Any>()
    hashMap["user"] = "ColaBlack"
    val latest: MutableMap<String, Any> = HashMap()
    hashMap["latestProduct"] = latest
    latest["url"] = "https://github.com/ColaBlack"
    latest["name"] = "No CRUD"
    // 获取模板文件
    val template = config.getTemplate("demo.ftl")
    val out = OutputStreamWriter(File("src/main/java/edu/zafu/generated/demo.java").outputStream())
    // 输出渲染后的内容
    template.process(hashMap, out)
    // 关闭输出流
    out.close()
}
```

对应的模版文件`demo.ftl`如下：

```text
package edu.zafu.generated;

public class demo {
    public static void main(String[] args) {
        System.out.println("Hello ${user}");
        System.out.println("The latest product is ${latestProduct.name}");
        System.out.println("You can find it at ${latestProduct.url}");
    }
}

```

将模版文件放入`src/main/resources/templates`目录下，运行`main`
函数，将生成的Java文件输出到`src/main/java/edu/zafu/generated/demo.java`文件中。

### 3.编写命令行picocli demo

参考官方文档，编写一个命令行demo，实现一个简单的`ls`命令。

```kotlin
import picocli.CommandLine
import picocli.CommandLine.*
import java.io.File
import kotlin.system.exitProcess

/**
 * ls命令demo
 *
 * @author ColaBlack
 */
// 定义一个dir命令，name为ls，description为该命令的描述信息，mixinStandardHelpOptions为true，表示该命令需要自动生成help选项
@Command(name = "dir", description = ["列出当前目录的目录结构"], mixinStandardHelpOptions = true)
// 定义一个dir类，继承Runnable接口，实现run方法，用于执行ls命令，其中Callable的泛型int表示call方法的返回值类型
class Dir : Runnable {

    // 该参数在命令行中指定，索引为0，description为描述信息
    @Parameters(index = "0", description = ["要列出的目录路径"])
    var path: String? = null // 定义一个path变量，用于接收命令行参数

    // 该选项缩写为-a，全称为--all，description为描述信息
    @Option(names = ["-a", "--all"], description = ["显示所有文件，包括隐藏文件"])
    var showHidden = false // 定义一个showHidden变量，用于接收-a选项，是否显示隐藏文件

    // 用户执行命令时，会调用run方法
    override fun run() {
        if (path == null) {
            println("文件路径不能为空")
            return
        }
        val file = File(path!!)
        if (!file.exists()) {     // 判断目录是否存在
            println("文件路径不存在: $path")
            return
        }
        if (!file.isDirectory) {  // 判断是否为目录
            println("$path 不是一个目录")
            return
        }
        val files = file.listFiles()
        if (files == null) {       // 判断目录是否为空
            println("目录为空")
            return
        }
        for (item in files) {
            if (showHidden || !item.name.startsWith(".")) {
                println(item.name)
            }
        }
    }
}

fun main(args: Array<String>) {
    val exitCode = CommandLine(Dir()).execute(*args)    // 执行命令
    exitProcess(exitCode)    // 退出程序
}
```

运行`main`函数并设置参数就可以使用CLI

### 4.编写需要制作成模版的代码

编写需要制作成模版的代码，例如本项目的controller代码：

```java
```

### 5.编写FreeMarker模版

将原代码中可以被参数化的部分用`${}`包裹起来就得到了模版，例如：

```java
```

### 6.制作CLI工具

完整代码参考：[No CRUD项目](https://github.com/ColaBlack/NoCRUD)
