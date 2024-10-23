智谱AI java SDK调用

参考:[智谱ai官方文档](https://open.bigmodel.cn/dev/howuse/introduction)

1.安装依赖

在maven 的pom.xml中输入

    <dependency>
        <groupId>cn.bigmodel.openapi</groupId>
        <artifactId>oapi-java-sdk</artifactId>
        <version>release-V4-2.3.0</version>
    </dependency>

2.编写通用AI调用工具类

    package edu.zafu.teaai.utils;
    
    import com.zhipu.oapi.ClientV4;
    import com.zhipu.oapi.Constants;
    import com.zhipu.oapi.service.v4.model.*;
    import edu.zafu.teaai.constant.AiConfig;
    import io.reactivex.Flowable;
    
    import java.util.ArrayList;
    import java.util.List;
    
    /**
     * AI调用模块
     *
     * @author ColaBlack
     */
    public class AiUtils {
    
        /**
         * 业务ID模版
         */
        private static final String REQUEST_ID_TEMPLATE = "teaAI-request-%s";
    
    
        /**
         * AI调用客户端
         */
        private static final ClientV4 CLIENT = new ClientV4.Builder(AiConfig.API_KEY).build();
    
        /**
         * 调用AI接口(同步)
         *
         * @param prompt 提示词
         * @return AI返回的答案
         */
        public static String aiCaller(String prompt) {
            List<ChatMessage> messages = new ArrayList<>();
            ChatMessage chatMessage = new ChatMessage(ChatMessageRole.USER.value(), prompt);
            messages.add(chatMessage);
            String requestId = String.format(REQUEST_ID_TEMPLATE, System.currentTimeMillis());
            ChatCompletionRequest chatCompletionRequest = ChatCompletionRequest.builder()
                    .model(AiConfig.MODEL_NAME)
                    .stream(Boolean.FALSE)
                    .invokeMethod(Constants.invokeMethod)
                    .messages(messages)
                    .requestId(requestId)
                    .build();
            ModelApiResponse invokeModelApiResp = CLIENT.invokeModelApi(chatCompletionRequest);
            return invokeModelApiResp.getData().getChoices().get(0).getMessage().getContent().toString();
        }
    
        /**
         * 调用AI接口(SSE)
         *
         * @author ColaBlack
         */
        public static Flowable<ModelData> aiCallerFlow(String prompt) {
            List<ChatMessage> messages = new ArrayList<>();
            ChatMessage chatMessage = new ChatMessage(ChatMessageRole.USER.value(), prompt);
            messages.add(chatMessage);
            String requestId = String.format(REQUEST_ID_TEMPLATE, System.currentTimeMillis());
            ChatCompletionRequest chatCompletionRequest = ChatCompletionRequest.builder()
                    .model(AiConfig.MODEL_NAME).
                    stream(Boolean.TRUE).
                    invokeMethod(Constants.invokeMethod)
                    .messages(messages)
                    .requestId(requestId)
                    .build();
            ModelApiResponse invokeModelApiResp = CLIENT.invokeModelApi(chatCompletionRequest);
            return invokeModelApiResp.getFlowable();
        }
    }
    

3.配置信息

将代码中的AiConfig.MODEL_NAME替换为要使用的模型名称，AiConfig.API_KEY替换为你的API_KEY

4.调用AI

如果要同步调用AI，那就直接将全部的提示词传入aiCaller方法，耐心等待即可返回结果。

这里消息的生产速度往往大于消费速度，因此可以考虑介入消息队列MQ。

同步调用如果一直让用户长时间等待用户体验不好，可以使用流式调用。

将全部的提示词传入aiCallerFlow方法得到一个Flowable对象。

为了将AI响应的结果传给前端，可以采用轮询、SSE、WebSocket等方式。这里选择使用SSE将响应由服务端推送给前端。

这里的代码是传入全部的提示词，返回一个SseEmitter对象给前端，方便前端使用。

    public SseEmitter generateQuestionSSE(String prompt) {
            // 建立 SSE 连接对象，0 表示不超时
            SseEmitter emitter = new SseEmitter(0L);
            // AI 生成的流式响应结果
            Flowable<ModelData> modelDataFlowable = AiUtils.aiCallerFlow(prompt);
        
            modelDataFlowable
                // 指定观察者的线程池
                .observeOn(Schedulers.io())
                // 从智谱的响应中获取数据
                .map(chunk -> chunk.getChoices().get(0).getDelta().getContent())
                // 去掉响应中多余的空格
                .map(message -> message.replaceAll("\\s", ""))
                // 去掉为空字符串的响应
                .filter(StringUtils::isNotBlank)
                .flatMap(message -> {
                // 将字符串转换为字符数组以便后续业务处理
                List<Character> charList = new ArrayList<>();
                for (char c : message.toCharArray()) {
                    charList.add(c);
                }
                return Flowable.fromIterable(charList);
            })
                .doOnNext(c -> {
                // 进行业务处理
                // 然后发送结果，这里为了演示直接不进行处理将给一个字符c都全部如实发送
                emitter.send(c);
            })
                //当AI响应完毕时关闭SSE
                .doOnComplete(emitter::complete)
                // 订阅AI响应流
                .subscribe();
            return emitter;
        }

4.前端接受SSE消息

这里选择使用原生的方式接收。

    const handleSSE = async () => {
      // 发送sse请求
      const eventSource = new EventSource(REQUEST_URL) //传入请求地址
    
      // 按项目给予用户生产中提示
      alert('生成中，请稍后')
    
      // 开始监听sse消息
      eventSource.onmessage = (event) => {
        //解析后端推送的消息，这里以后端传JSON字符串为例
        const res = JSON.parse(event.data)
        //进行处理，以追加到form.test为例
        form.value.test = [...(form.value.test || []), res]
      }
      //处理报错和停止接受
      eventSource.onerror = (event) => {
          //正常停止接收
        if (event.eventPhase === EventSource.CLOSED) {
          eventSource.close()
          alert('生成结束')
        } else {
           //报错记录日志
          eventSource.close()
         console.error('生成失败')
        }
      }
    }