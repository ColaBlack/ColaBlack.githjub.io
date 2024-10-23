java操作邮箱 - 邮箱发送验证码 -redis分布式缓存 -redisson分布式缓存

参考:java操作163邮箱

本文以163邮箱为例，介绍如何用java发送邮箱。

1.获取邮箱授权码

进入163邮箱-设置-POP3/SMTP/IMAP-开启POP3/SMTP服务

记录得到的授权码

2.安装依赖

    // https://mvnrepository.com/artifact/jakarta.activation/jakarta.activation-api
    implementation("jakarta.activation:jakarta.activation-api:2.1.3")
    // https://mvnrepository.com/artifact/org.apache.commons/commons-email
    implementation("org.apache.commons:commons-email:1.6.0")
    

注意，参考文章年代有些久远，有几个库已经合并更新换了新的名字。

3.编写通用邮件发送工具类

    package edu.zafu;
    
    
    import org.apache.commons.mail.EmailException;
    import org.apache.commons.mail.SimpleEmail;
    
    /**
     * 邮件工具类
     *
     * @author ColaBlack
     */
    public class MailUtils {
    
        /**
         * 发送邮件
         *
         * @param targetEmail 目标用户邮箱
         * @param header      邮件的标题
         * @param message     要发送的消息
         */
        public static void sendEmail(String targetEmail, String header, String message) {
            try {
                // 创建邮箱对象
                SimpleEmail mail = new SimpleEmail();
                // 设置发送邮件的服务器，以163邮箱为例
                mail.setHostName("smtp.163.com");
                // 输入发送邮件的邮箱号+授权码
                mail.setAuthentication("用于发送的邮箱号", "授权码");
                // 注意：一个邮箱账号可能有多个邮箱，要注意这个区分。
                // 如QQ邮箱就支持同一个邮箱账号持有@qq.com @foxmail.com两个邮箱
                // 发送邮件 "你的邮箱号"+"发送时用的昵称"
                mail.setFrom("用于发送邮箱的邮箱号", "昵称");
                // 使用SSL安全链接
                mail.setSSLOnConnect(true);
                // 接收用户的邮箱
                mail.addTo(targetEmail);
                // 邮件的主题(标题)
                mail.setSubject(header);
                // 邮件的内容
                mail.setMsg(message);
                // 发送
                mail.send();
            } catch (EmailException e) {
                // 邮件发送失败，记录日志(此处应该换成项目统一的日志记录方式)
                System.out.println("邮件发送失败：" + e.getMessage());
            }
        }
    }



4.调用工具类——以发验证码为例

编写发送验证码工具类

    package edu.zafu;
    
    import java.util.Random;
    
    /**
     * 发送验证码工具类
     *
     * @author ColaBlack
     */
    public class SendCode {
    
        /**
         * 发送验证码
         *
         * @param targetEmail 目标邮箱
         */
        public static void send(String targetEmail) {
            // 生成验证码
            int code = new Random().nextInt(899999) + 100000;
            // 发送验证码
            MailUtils.sendEmail(targetEmail, "验证码", "您的验证码为:" + code + "(1分钟内有效)");
        }
    }
    

可以使用如下代码测试

    package edu.zafu;
    
    /**
     * 测试邮箱发送
     *
     * @author ColaBlack
     */
    public class TestMail {
        public static void main(String[] args) {
            SendCode.send("你的第二个邮箱");
        }
    }
    

5.引入缓存机制

上面的验证码用户可以通过疯狂发送请求而恶意消耗邮箱发送资源，我们也没有让验证码及时过期。

可以引入缓存机制解决，在上文Caffeine本地缓存和缓存雪崩，缓存击穿，缓存穿透中我们介绍了如何利用caffeine实现本地缓存，本文就选择用 Redis 实现分布式缓存。

6.安装Redisson

为了在java中操作redis需要安装依赖

    // https://mvnrepository.com/artifact/org.redisson/redisson
    implementation("org.redisson:redisson:3.37.0")
    

7.调整验证码工具类

    package edu.zafu;
    
    import org.redisson.Redisson;
    import org.redisson.api.RMapCache;
    import org.redisson.api.RedissonClient;
    import org.redisson.config.Config;
    
    import java.util.Random;
    import java.util.concurrent.TimeUnit;
    
    /**
     * 发送验证码工具类
     *
     * @author ColaBlack
     */
    public class SendCode {
    
        /**
         * 发送验证码
         *
         * @param targetEmail 目标邮箱
         */
        public static void send(String targetEmail) {
            Config config = new Config();
            config.useSingleServer() // 使用单机模式
                    // 设置要连接的数据库
                    .setDatabase(0)
                    // 设置redis服务器地址
                    .setAddress("redis://127.0.0.1:6379")
                    // 设置redis密码
                    .setPassword("password");
            // 创建Redisson客户端
            RedissonClient redisson = Redisson.create(config);
            // 获取缓存的map
            RMapCache<Object, Object> codeCache = redisson.getMapCache("code");
            // 判断目标邮箱是否已存在验证码
            if (codeCache.containsKey(targetEmail)) {
                // 验证码已存在，直接返回
                return;
            }
            // 验证码不存在，生成验证码并存入缓存
            // 生成验证码
            int code = new Random().nextInt(899999) + 100000;
            // 存入缓存,设置1分钟过期
            codeCache.put(targetEmail, code, 1, TimeUnit.MINUTES);
            // 发送验证码
            MailUtils.sendEmail(targetEmail, "验证码", "您的验证码为:" + code + "(1分钟内有效)");
            // 校验验证码是否正确时也从redis中拿去对应邮箱的验证码，这样就能实现验证码的超时过期
        }
    }
    

关于使用缓存机制可能引发的缓存击穿、缓存雪崩、缓存穿透的问题请参考上文Caffeine本地缓存和缓存雪崩，缓存击穿，缓存穿透


