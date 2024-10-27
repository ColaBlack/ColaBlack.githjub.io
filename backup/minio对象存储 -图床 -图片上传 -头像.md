# minio对象存储 -图床 -图片上传 -头像

参考：

1,[MINIO在java中的使用](https://blog.csdn.net/weixin_43676950/article/details/118938099)

2.[MinIO Linux官方文档](https://min.io/docs/minio/linux/index.html)

### 1.利用1panel安装minio

图片上传等服务依赖于对象存储服务，本文就以开源对象储存minio为例简单介绍。

推荐使用linux版本的minio，只需在1panel应用商城傻瓜式安装即可。

记得记好你的root账户用户名和密码，另外minio不支持弱密码弱用户名！

1panel基于容器部署，记得开启外部访问权限。

### 2.安装java sdk

```xml
<!-- https://mvnrepository.com/artifact/io.minio/minio -->
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.13</version>
</dependency>

```

或

```kotlin
// https://mvnrepository.com/artifact/io.minio/minio
implementation("io.minio:minio:8.5.13")

```

### 3.配置minio连接

在applicantion.yml添加如下配置。

```yaml
minio:
  endpoint: http://你的minio主机:9000
  accesskey: 你的ak
  secretkwy: 你的sk
```

注意，minio默认9000端口是上传文件的端口，9001是其web控制台端口，ak‘和sk都可以在那里获取。



然后新增配置类：

先配置如何在applicantion.yml中读取配置信息

```java
package edu.zafu.teaai.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * 获取minio的配置信息
 *
 * @author ColaBlack
 */
@Data
@Component
@ConfigurationProperties(prefix = "minio")
// 此处说明了要读取minio.*的配置信息
public class MinioProps {

    /**
     * minio的url
     */
    private String endpoint;

    /**
     * minio的ak
     */
    private String accesskey;

    /**
     * minio的sk
     */
    private String secretkwy;
}


```

然后是真正着手配置minio

```java
package edu.zafu.teaai.config;


import io.minio.MinioClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.Resource;

/**
 * minio配置类
 *
 * @author ColaBlack
 */
@Configuration
public class MinioConfig {

    @Resource
    private MinioProps props;

    /**
     * 获取 Minio客户端对象
     *
     * @return MinioClient
     */
    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
                .endpoint(props.getEndpoint())
                .credentials(props.getAccesskey(), props.getSecretkwy())
                .build();
    }
}

```



最后是在minio上传文件的工具类。

这也是重点中的重点。

对象存储以存储桶为基本单位，你可以粗略得认为存储桶就是一个“数据库”，下面来介绍一下如何创建存储桶。

当然，事实上这个存储桶应该由管理员在minio web界面上创建而不是用java程序创建。不过无妨，这里也讲讲。

我们要通过刚刚配置的minio客户端对minio进行操作，所以要先通过依赖注入获取客户端。

```java
@Resource
private MinioClient client;
```

然后再写一个方法创建指定名称的存储桶。

```java
    /**
     * 创建桶
     *
     * @param bucketName 桶名称
     */
    public void createBucket(String bucketName) {
        boolean found = client.bucketExists(
                BucketExistsArgs.builder().
                        bucket(bucketName)
                        .build()
        );
        if (!found) {
            client.makeBucket(
                    MakeBucketArgs.builder()
                            .bucket(bucketName)
                            .build());
        }
    }
```

第一段代码是通过客户端的bucketExists方法判断指定名称的存储桶是否存在。

但是我们还得给他传一个查询参数才行，这就是括号里面传的参数了。

通过

```java
BucketExistsArgs.builder().bucket(bucketName).build()
```

即可得到一个查询桶名称为字符串bucketName内容的桶。

如果存在这样的桶则found=true，反之为false。

下一段则是说，如果桶不存在，那就通过client.makeBucket创建桶。

当然，你可能会发现idea报error说方法可能抛出两个异常，你可以通过try catch手动捕获异常进行处理，也可以加上lombok的@SneakyThrows交由lombok自动处理。

最后上传文件的工具方法就变成了

```java
	@Resource
    private MinioClient client;

    /**
     * 上传文件
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @param stream     输入流
     * @param fileSize   文件大小
     */
    @SneakyThrows
    public void putObject(String bucketName, String objectName, InputStream stream, Long fileSize) {
        client.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(stream, fileSize, -1)
                        .build());
    }
```



删除桶啊什么的自己看文档吧，实际上暂时只有上传文件和获取文件的URL会用到。

上传文件则需要文件的输入流stream等信息

```java
    /**
     * 上传文件
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @param stream     输入流
     * @param fileSize   文件大小
     * @param type       文件类型
     */
    @SneakyThrows
    public void putObject(String bucketName, String objectName, InputStream stream, Long fileSize, String type) {
        client.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(stream, fileSize, -1)
                        .contentType(type)
                        .build());
    }
```

需要注意的是，一般而言文件类型minio可以自己根据后缀名自己识别，在这里可以不给出。

常见的文件类型

> - `text/plain`: 文本文件（.txt）
> - `text/html`: HTML文件（.html, .htm）
> - `application/json`: JSON文件（.json）
> - `application/xml`: XML文件（.xml）
> - `image/jpeg`: JPEG图片（.jpg, .jpeg）
> - `image/png`: PNG图片（.png）
> - `image/gif`: GIF图片（.gif）
> - `application/pdf`: PDF文件（.pdf）
> - `application/zip`: ZIP文件（.zip）

所以最简单的上传方法如下：

```java
    /**
     * 上传文件
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @param stream     输入流
     * @param fileSize   文件大小
     */
    @SneakyThrows
    public void putObject(String bucketName, String objectName, InputStream stream, Long fileSize) {
        client.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(stream, fileSize, -1)
                        .build());
    }
```

还有一个需要的是获取文件的外链，也就是外部可以访问的URL信息

```java
    /**
     * 获取文件在minio在服务器上的外链
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @return 外链
     */
    @SneakyThrows
    public String getObjectUrl(String bucketName, String objectName) {
        return client.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                        .method(Method.GET)
                        .bucket(bucketName)
                        .object(objectName)
                        .build());
    }
```

其他的方法可以自己查询文档获取，使用率并不高。

最终整个工具类的代码如下

```java
package edu.zafu.teaai.utils;


import io.minio.GetPresignedObjectUrlArgs;
import io.minio.MinioClient;
import io.minio.PutObjectArgs;
import io.minio.http.Method;
import lombok.SneakyThrows;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.io.InputStream;

/**
 * minio操作类
 *
 * @author ColaBlack
 */
@Component
public class MinioUtils {

    @Resource
    private MinioClient client;

    /**
     * 上传文件
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @param stream     输入流
     * @param fileSize   文件大小
     */
    @SneakyThrows
    public void putObject(String bucketName, String objectName, InputStream stream, Long fileSize) {
        client.putObject(
                PutObjectArgs.builder()
                        .bucket(bucketName)
                        .object(objectName)
                        .stream(stream, fileSize, -1)
                        .build());
    }


    /**
     * 获取文件在minio在服务器上的外链
     *
     * @param bucketName 桶名称
     * @param objectName 文件名
     * @return 外链
     */
    @SneakyThrows
    public String getObjectUrl(String bucketName, String objectName) {
        return client.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                        .method(Method.GET)
                        .bucket(bucketName)
                        .object(objectName)
                        .build());
    }

}

```



### 4.编写工具类和service层代码

本文以头像上传为例进行简介。

首先编写service层接口代码

```java
package edu.zafu.teaai.service;

import edu.zafu.teaai.model.dto.file.UploadFileRequest;
import org.springframework.web.multipart.MultipartFile;

import javax.servlet.http.HttpServletRequest;

/**
 * minio文件服务接口
 *
 * @author ColaBlack
 */
public interface MinioService {
    /**
     * 上传文件
     *
     * @param file              上传的文件
     * @param uploadFileRequest 上传文件请求
     * @param request           请求对象
     * @return 上传结果
     */
    String uploadImage( MultipartFile file, UploadFileRequest uploadFileRequest, HttpServletRequest request);

    /**
     * 验证文件
     *
     * @param file 文件
     */
    void validFile(MultipartFile file);
}

```

接着在编写实现类时会遇到几个常量， 在此提前给出。

```java
/**
 * 文件上传相关常量
 *
 * @author ColaBlack
 */
public interface FileConstant {
    /**
     * 上传文件最大大小1MB
     */
    int MAX_FILE_SIZE = 1048576;

    /**
     * 允许上传的图片类型
     */
    List<String> ALLOW_IMAGE_TYPES = Arrays.asList("jpg", "jpeg", "png", "svg", "webp", "bmp");

}
```

这两个常量都是用于验证头像信息的。

然后是编写实现类，首先是验证图片信息的方法。

```java
    /**
     * 校验文件
     *
     * @param file 文件
     */
    @Override
    public void validFile(MultipartFile file) {
        ThrowUtils.throwIf(ObjectUtils.isEmpty(file), ErrorCode.PARAMS_ERROR, "文件不能为空");
        // 文件大小
        long fileSize = file.getSize();
        if (fileSize > FileConstant.MAX_FILE_SIZE) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件大小不能超过 1M");
        }
        // 文件后缀
        String fileSuffix = FileUtil.getSuffix(file.getOriginalFilename());
        if (!FileConstant.ALLOW_IMAGE_TYPES.contains(fileSuffix)) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件类型错误");
        }
    }
```

主要是对文件大小和文件类型进行了限制。

然后是重头戏，操作minio上传文件。

在上传文件前需要根据业务进行一些检查，例如只有已登录用户才能上传头像等等，在此只调用刚刚写的validFile方法。

```java
// 校验文件
validFile(file);
```

然后是对用户上传的头像进行重命名，避免两个不同的用户上传了同一名字的图片导致异常覆盖。

本文以：业务名\_当前时间戳\_随机uuid\_文件原始名称为重命名模版。

```java
//重命名文件
FileUploadBizEnum fileUploadBizEnum = FileUploadBizEnum.getEnumByValue(biz);
String fileName = biz + "_" + System.currentTimeMillis() + "_" + UUID.randomUUID() + "_" + file.getOriginalFilename();
```

然后调用minioUtils的方法上传文件即可。

```java
//使用minio上传文件
minioUtils.putObject(FileConstant.BUCKET_NAME, fileName, file.getInputStream(), file.getSize());
//获取外链
return minioUtils.getObjectUrl(FileConstant.BUCKET_NAME, fileName);
```

最后minio服务实现类的代码就是

```java
package edu.zafu.teaai.service.impl;

import cn.hutool.core.io.FileUtil;
import edu.zafu.teaai.common.ErrorCode;
import edu.zafu.teaai.common.exception.BusinessException;
import edu.zafu.teaai.common.exception.ThrowUtils;
import edu.zafu.teaai.constant.FileConstant;
import edu.zafu.teaai.constant.UserConstant;
import edu.zafu.teaai.service.MinioService;
import edu.zafu.teaai.utils.MinioUtils;
import lombok.SneakyThrows;
import org.apache.commons.lang3.ObjectUtils;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import java.util.UUID;

/**
 * Minio文件服务实现类
 *
 * @author ColaBlack
 */
@Service
public class MinioServiceImpl implements MinioService {

    @Resource
    private MinioUtils minioUtils;

    /**
     * 上传文件
     *
     * @param file    上传的文件
     * @param biz     业务类型
     * @param request 请求对象
     * @return 上传结果
     */
    @SneakyThrows
    @Override
    public String uploadImage(MultipartFile file, String biz, HttpServletRequest request) {
        // 校验文件
        validFile(file);

        //登录用户才能上传文件
        Object attribute = request.getSession().getAttribute(UserConstant.USER_LOGIN_STATE);
        ThrowUtils.throwIf(ObjectUtils.isEmpty(attribute), ErrorCode.NOT_LOGIN_ERROR, "用户未登录");

        //重命名文件
        String fileName = biz + "_" + System.currentTimeMillis() + "_" + UUID.randomUUID() + "_" + file.getOriginalFilename();

        //使用minio上传文件
        minioUtils.putObject(FileConstant.BUCKET_NAME, fileName, file.getInputStream(), file.getSize());
        //获取外链
        return minioUtils.getObjectUrl(FileConstant.BUCKET_NAME, fileName);
    }

    /**
     * 校验文件
     *
     * @param file 文件
     */
    @Override
    public void validFile(MultipartFile file) {
        ThrowUtils.throwIf(ObjectUtils.isEmpty(file), ErrorCode.PARAMS_ERROR, "文件不能为空");
        // 文件大小
        long fileSize = file.getSize();
        if (fileSize > FileConstant.MAX_FILE_SIZE) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件大小不能超过 1M");
        }
        // 文件后缀
        String fileSuffix = FileUtil.getSuffix(file.getOriginalFilename());
        if (!FileConstant.ALLOW_IMAGE_TYPES.contains(fileSuffix)) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件类型错误");
        }
    }
}

```

最后提醒一下，别忘了service层实现类要有Service注解。

### 5.编写controller层代码

在编写前需要先针对项目的业务编写枚举类

```java
package edu.zafu.teaai.model.enums;

import lombok.AllArgsConstructor;
import lombok.Getter;
import org.apache.commons.lang3.ObjectUtils;

import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 文件上传业务类型枚举
 *
 * @author ColaBlack
 */
@Getter
@AllArgsConstructor
public enum FileUploadBizEnum {

    /**
     * 用户头像上传
     */
    USER_AVATAR("用户头像", "user_avatar"),

    /**
     * 业务类型名称
     */
    private final String text;

    /**
     * 业务类型值
     */
    private final String value;


    /**
     * 获取值列表
     */
    public static List<String> getValues() {
        return Arrays.stream(values()).map(item -> item.value).collect(Collectors.toList());
    }

    /**
     * 根据 value 获取枚举
     */
    public static FileUploadBizEnum getEnumByValue(String value) {
        if (ObjectUtils.isEmpty(value)) {
            return null;
        }
        for (FileUploadBizEnum anEnum : FileUploadBizEnum.values()) {
            if (anEnum.value.equals(value)) {
                return anEnum;
            }
        }
        return null;
    }
}

```

在controller层，先注入minioService，然后进行简单的校验

```java
    @Resource
    private MinioService minioService;

    @PostMapping("/upload/avatar")
    public BaseResponse<String> uploadAvatar(@RequestPart("file") MultipartFile file,
                                             UploadFileRequest uploadFileRequest, HttpServletRequest request) {

        //登录用户才能上传文件
        Object attribute = request.getSession().getAttribute(UserConstant.USER_LOGIN_STATE);
        ThrowUtils.throwIf(ObjectUtils.isEmpty(attribute), ErrorCode.NOT_LOGIN_ERROR, "用户未登录");

        // 校验业务类型
        String biz = uploadFileRequest.getBiz();
        FileUploadBizEnum fileUploadBizEnum = FileUploadBizEnum.getEnumByValue(biz);
        ThrowUtils.throwIf(!Objects.equals(fileUploadBizEnum, FileUploadBizEnum.USER_AVATAR), ErrorCode.PARAMS_ERROR, "上传业务类型不正确");

        // 上传文件
        String res = minioService.uploadImage(file, biz, request);
        return ResultUtils.success(res);
    }
```

然后调用service层代码即可。

