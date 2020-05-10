---
title:  "一次微服务重构总结 "
date:   2020-05-10 23:48:05 +0800
tag: [Java, 微服务, 重构]
---

# 一次微服务重构总结 

## 前言

微信服务是我司内部的一个微服务，主要是接入微信官方的一些能力，进行封装以后提供给内部其他业务进行使用。

该服务诞生于 2016 年，至今也有些年头了，为了符合公司新的项目结构，需要对目录进行一定的改造，在对项目进行一个简单的了解以后，我感觉它已经有点 “腐朽” 的味道，于是就将目录改造的目标提升为项目重构。

PS：虽然你看它不顺眼，但是它就是能运行

![](img/run.gif)



> 重构很像是在整理代码，你所做的就是让所有东西回到应处的位置上。
>
> 代码结构的流失是累积性的。越难看出代码所代表的设计意图，就越难保护其中设计，于是该设计就腐败得越快。
>
> 《重构》



## 重构

### 分层结构的流失

该服务采用的是传统的分层架构，但实际结构已经演化成如下图这样了

![image-20200509151928701](img/layer-intersaction.svg)



几乎看不出组件之间的层次了，单向的依赖关系变成了两两的交集关系，这是一个典型的累积型代码结构流失。

为了改变这样的情况，第一步就是先重新定义层次和组件依赖关系

先划分组件和各自的职责，组件划分保留原有设计，分为 controller、service、model 和 wx-api 四块。

- **controller** 负责提供 REST API
- **service** 负责核心业务逻辑
- **model** 提供与 DB 交互的能力
- **wx-api** 提供于微信官方服务交互的能力

然后就是明确依赖规则

- 依赖方向只允许单向流动
- 禁止跨层依赖
- model 和 wx-api 放到同一层，但是禁止相互依赖

按照以上定义，最终可以得到下图所示的结构

![](img/layer-arch.svg)





### 不恰当的模型共用

在前面一节中，重新划分了组件层次和依赖，但是仍然有一个问题,  从最下层的 **WX-Api** 到最上层的 **Controller**，有公用了一部分对象模型。

![](img/same-model.svg)

也许很多人这时候就有疑问了，如果 controller 的接口参数和响应和微信的某个接口一样，直接共用一套对象不是即节省时间，又避免了无用的类定义吗？

这就涉及到一个变化原因的问题了

- WX-Api 的变化速率受微信官方 API 迭代的影响
- Controller / Service 的变化速率主要是受业务的影响

组件之间的变化原因不同，那么变化的速率自然也不尽相同。而此时几个组件共用一套模型对象，势必会产生冲突，所以模型对象应该独立开来



![image-20200510002006737](img/dependent-model.svg)



### 隐式的循环依赖

什么是循环依赖就不再解释了，这在代码中是一个非常典型的坏味道。

在该服务中，循环依赖是由于继承关系而隐式造成的

以 `ThirdService`  和  `TokenService` 为例

- ThirdService 继承了 BaseService,  而 BaseService 依赖了 TokenService
- TokenService 依赖了 ThirdService

参见下图

![image-20200509150025346](img/image-20200509150025346.png)



为什么要这样实现呢？

在 BaseService 中有这样一段代码，这是造成循环依赖的罪魁祸首

```java
/**
* 该方法主要是判断调用微信接口时响应的 code 是否在 TOKEN_ERROR_CODES 中
* 如果是的话就重新刷新 token
* （TOKEN_ERROR_CODES 是一个 List<Integer>）
* 
*/
if (TOKEN_ERROR_CODES.contains(code)) {
  	tokenServices.refreshToken(appId);
  	throw new DomainException(DomainError.with(code, ErrorCode.getMsg(code)));
}
```

TokenService 和 BaseService 都是在 service 组件中，属于同一层，应该避免相互依赖。

为了解决循环依赖，我定义了一个单独的异常

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class WeChatTokenErrorException {
  
  private WeixinAppId auth;
  // ...
}
```

然后将前面的代码改成这样

```java
if (TOKEN_ERROR_CODES.contains(code)) {
  	throw new WeChatTokenException(appId);
}
```

最后在 controller 组件中定义 Advice

```java
@ControllerAdvice
public class ApiExceptionHandler {
  
  @Autowired
  private TokenService tokenService;
  
  
    @ExceptionHandler(Throwable.class)
    @ResponseBody
    public JsonResult handleAllException(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Throwable error) {
        if (error instanceof WeChatTokenErrorException) {
            tokenServices.refreshToken(exception.getWeixinAppId());
        		return JsonResult.error();
        }

        // 忽略其他代码
    }
}
```

这样就解决了循环依赖的问题，同时还遵循了最开始定义的分层规约





### 偷懒的方法签名

在面向对象编程中方法的定义也是一种抽象，但是不合适的抽象等于没有抽象。

而在该服务中很多方法定义就明显是不合适的抽吸那个，下面是一段来自于 WX-Api 组件内的代码

```java
@POST(WxServiceConfig.URL_SEND)
Call<ApiResult> sendMessage(@Query("access_token") String token,
                            @Body JSONObject object);
                                
@POST(WxServiceConfig.URL_SEND_TEMPLATE)
Call<ApiResult> sendTemplateMessage(@Query("access_token") String token,
                                    @Body JSONObject object);

```

- sendMessage 是接入的微信客服消息发送接口
- sendTemplate 是接入的微信模板消息发送接口

注意两个接口的参数都是 JSONObject 对象，这对调用方来说绝对是一个灾难，因为调用者只知道是个 JSON，但却不知道要什么样的 JSON。

最后将其改成这样

```java
@POST(WxServiceConfig.URL_SEND)
Call<ApiResponse> sendMessage(@Query("access_token") String token, @Body VoiceMessage message);

@POST(WxServiceConfig.URL_SEND)
Call<ApiResponse> sendMessage(@Query("access_token") String token, @Body ImageMessage message);

@POST(WxServiceConfig.URL_SEND)
Call<ApiResponse> sendMessage(@Query("access_token") String token, @Body ArticleMessage message);

@POST(WxServiceConfig.URL_SEND_TEMPLATE)
Call<ApiResponse> sendTemplateMessage(@Query("access_token") String token,
                                          @Body TemplateMessageRequest request);
```





## 总结

重构前和重构后，该服务的**行为价值**并没有显著的改变，都能正常运行并提供相同的功能。

但是该项目的**架构价值**却有了质的提升，让渐渐流失的设计重新得以维持。

当然，这离我想象中的设计还有一定的距离，所以重构没有完成时，始终都是进行时。

