# [SpringBoot项目的国际化流程](https://www.cnblogs.com/Marktowin/p/19511993 "发布于 2026-01-21 15:15")

在 Spring Boot 项目已经开发完成后，想要实现国际化（i18n），让所有提示信息（后端返回的错误消息、成功消息、异常信息、枚举描述等）支持多语言，处理流程如下：

### 1\. 创建国际化资源文件（messages.properties）

在 src/main/resources 目录下（新建 i18n 子目录），创建以下文件：

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

basic

1 src/main/resources/
2 └── i18n/
3     ├── messages.properties           # 默认语言（通常是中文或英文）
4     ├── messages\_zh\_CN.properties     # 简体中文
5     ├── messages\_en\_US.properties     # 美式英文
6     ├── messages\_zh\_TW.properties     # 繁体中文（可选）
7     └── messages\_ja.properties        # 日语（可选）

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

messages.properties（中文示例）：

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

basic

1 \# 通用提示
2 success=操作成功
3 error.system\=系统异常，请稍后重试
4 error.notfound=资源不存在
5 
6 \# 业务提示
7 user.login.success=登录成功
8 user.login.fail=用户名或密码错误
9 user.notfound=用户不存在

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

messages\_en\_US.properties（英文示例）：

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

basic

1 success=Operation successful
2 error.system\=System error, please try again later
3 error.notfound=Resource not found
4 
5 user.login.success=Login successful
6 user.login.fail=Username or password is incorrect
7 user.notfound=User not found

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

**注意**：

-   文件名必须以 messages 开头（Spring Boot 默认查找规则）。
-   所有提示统一放在 i18n 目录下，方便管理。

### 2\. 配置 application.yml（或 application.properties）

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

basic

1 \# application.yml
2 spring:
3   messages:
4     basename: i18n/messages          # 资源文件基础名（支持通配符）
5     encoding: UTF-8                 # 必须是 UTF-8
6     fallback-to\-system\-locale: false # 推荐设置为 false（找不到对应语言时不回退到系统默认Locale）
7     use-code-as-default-message: true # 找不到key时直接返回code（调试方便，上线可改为false）

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

basic

1 \#  properties
2 spring.messages.basename=i18n/messages
3 spring.messages.encoding=UTF-8
4 spring.messages.fallback-to\-system\-locale=false
5 spring.messages.use-code-as-default-message=true

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

### 3\. 自定义 LocaleResolver 和 LocaleChangeInterceptor（支持前端传参切换语言）

Spring Boot 默认使用 **AcceptHeaderLocaleResolver**（根据请求头 Accept-Language 自动识别），生产环境通常还需要支持通过参数或 cookie 切换语言。

使用 SessionLocaleResolver + LocaleChangeInterceptor：

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

aspectj

 1 @Configuration
 2 public class LocaleConfig implements WebMvcConfigurer {
 3 
 4     // 默认语言
 5     @Bean
 6     public LocaleResolver localeResolver() {
 7         SessionLocaleResolver localeResolver = new SessionLocaleResolver();
 8         localeResolver.setDefaultLocale(Locale.SIMPLIFIED\_CHINESE); // 默认简体中文
 9         return localeResolver;
10     }
11 
12     // 拦截器：支持 ?lang=zh\_CN 或 ?lang=en\_US 切换语言
13     @Bean
14     public LocaleChangeInterceptor localeChangeInterceptor() {
15         LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
16         lci.setParamName("lang"); // 前端传参名
17         return lci;
18     }
19 
20     @Override
21     public void addInterceptors(InterceptorRegistry registry) {
22         registry.addInterceptor(localeChangeInterceptor());
23     }
24 }

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

若优先从请求头识别，再支持参数切换，可以自定义 LocaleResolver（更灵活）。

### 4\. 在代码中使用 MessageSource 获取国际化消息

#### 方式一：注入 MessageSource

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

typescript

 1 @RestController
 2 @RequestMapping("/api")
 3 public class UserController {
 4 
 5     @Autowired
 6     private MessageSource messageSource;
 7 
 8     @GetMapping("/test")
 9     public ResponseEntity<String\> test(Locale locale) { // Locale 可选从参数注入
10         // 方式1：直接使用 locale 参数
11         String msg = messageSource.getMessage("user.login.success", null, locale);
12 
13         // 方式2：从 LocaleContextHolder 获取当前语言（推荐！）
14         String msg2 = messageSource.getMessage("user.login.success", null, LocaleContextHolder.getLocale());
15 
16         return ResponseEntity.ok(msg2);
17     }
18 }

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

#### 方式二：统一封装工具类

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

typescript

 1 @Component
 2 public class MessageUtils {
 3 
 4     private static MessageSource messageSource;
 5 
 6     @Autowired
 7     public void setMessageSource(MessageSource messageSource) {
 8         MessageUtils.messageSource = messageSource;
 9     }
10 
11     /\*\*
12      \* 获取国际化消息
13      \*/
14     public static String getMessage(String code, Object... args) {
15         return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
16     }
17 
18     public static String getMessage(String code) {
19         return getMessage(code, (Object) null);
20     }
21 }

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

使用方式：

basic

1 throw new BusinessException(MessageUtils.getMessage("user.notfound"));
2 // 或
3 return Result.error(MessageUtils.getMessage("error.system"));

### 5\. 全局异常处理中使用国际化

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

reasonml

 1 @RestControllerAdvice
 2 public class GlobalExceptionHandler {
 3 
 4     @ExceptionHandler(BusinessException.class)
 5     public Result<?> handleBusinessException(BusinessException e) {
 6         // 假设 BusinessException 里面存了 messageCode
 7         String message = MessageUtils.getMessage(e.getCode(), e.getArgs());
 8         return Result.error(message);
 9     }
10 
11     @ExceptionHandler(Exception.class)
12     public Result<?> handleException(Exception e) {
13         return Result.error(MessageUtils.getMessage("error.system"));
14     }
15 }

[](javascript:void\(0\); "复制代码")[![](//assets.cnblogs.com/images/copycode.gif)](//assets.cnblogs.com/images/copycode.gif)

### 6\. 支持参数占位符

basic

1 \# messages.properties
2 user.age.limit=年龄必须在 {0} 到 {1} 岁之间

basic

1 String msg = MessageUtils.getMessage("user.age.limit", 18, 60);
2 // 输出：年龄必须在 18 到 60 岁之间

### 7\. 常见最佳实践总结

| 处理点 | 处理方式 |
| --- | --- |
| 资源文件位置 | `src/main/resources/i18n/messages_*.properties` |
| 默认语言 | 简体中文（`Locale.SIMPLIFIED_CHINESE`） |
| 语言切换方式 | 请求头 `Accept-Language` + `?lang=zh_CN` |
| 消息获取方式 | 优先使用 `LocaleContextHolder.getLocale()` |
| 工具类 | 封装 `MessageUtils` 统一获取 |
| 异常消息 | 全部使用 code + MessageUtils 获取 |
| 找不到key | 建议 `use-code-as-default-message=false` |
| 编码 | 必须 `UTF-8` |

### 8\. 测试方法

-   浏览器开发者工具 → Network → 修改请求头 Accept-Language: en-US,en;q=0.9
-   或直接在 URL 后加 ?lang=en\_US

完成以上步骤，你的 Spring Boot 项目就实现了标准的国际化支持，所有提示信息都可以根据用户语言自动切换。

-   [1\. 创建国际化资源文件（messages.properties）](#tid-wRjTbC)
-   [2\. 配置 application.yml（或 application.properties）](#tid-J5KAMF)
-   [3\. 自定义 LocaleResolver 和 LocaleChangeInterceptor（支持前端传参切换语言）](#tid-B5brpH)
-   [4\. 在代码中使用 MessageSource 获取国际化消息](#tid-YA2B32)
-       [方式一：注入 MessageSource](#tid-SEQekt)
-       [方式二：统一封装工具类](#tid-JPaRFy)
-   [5\. 全局异常处理中使用国际化](#tid-7GXehn)
-   [6\. 支持参数占位符](#tid-yKecCh)
-   [7\. 常见最佳实践总结](#tid-pzxYiN)
-   [8\. 测试方法](#tid-StnPRA)

  

\_\_EOF\_\_

[![](https://cdn.cnblogs.com/gh/BNDong/Cnblogs-Theme-SimpleMemory@v2.1.0/dist/images/53abc64338825f4038d6.webp)](https://cdn.cnblogs.com/gh/BNDong/Cnblogs-Theme-SimpleMemory@v2.1.0/dist/images/53abc64338825f4038d6.webp)

-   **本文作者：** [Marktowin](https://www.cnblogs.com/Marktowin)
-   **本文链接：** [https://www.cnblogs.com/Marktowin/p/19511993](https://www.cnblogs.com/Marktowin/p/19511993)
-   **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://msg.cnblogs.com/msg/send/Marktowin)我。
-   **版权声明：** 除特殊说明外，转载请注明出处～\[知识共享署名-相同方式共享 4.0 国际许可协议\]
-   **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void\(0\);)】**一下。

分类: [Java](https://www.cnblogs.com/Marktowin/category/2442561.html), [SpringBoot](https://www.cnblogs.com/Marktowin/category/2491767.html)

标签: [Java](https://www.cnblogs.com/Marktowin/tag/Java/), [SpringBoot](https://www.cnblogs.com/Marktowin/tag/SpringBoot/), [后端](https://www.cnblogs.com/Marktowin/tag/%E5%90%8E%E7%AB%AF/)

免责声明：本内容来自平台创作者，博客园系信息发布平台，仅提供信息存储空间服务。

[好文要顶](javascript:void\(0\);)推荐该文

[关注我](javascript:void\(0\);)关注博主关注博主 [收藏该文](javascript:void\(0\);)收藏本文 [微信分享](javascript:void\(0\);)分享微信

[![](https://pic.cnblogs.com/face/1454713/20250827140434.png)](https://home.cnblogs.com/u/Marktowin/)

[Marktowin](https://home.cnblogs.com/u/Marktowin/)  
[粉丝 - 2](https://home.cnblogs.com/u/Marktowin/followers/) [关注 - 22](https://home.cnblogs.com/u/Marktowin/followees/)  

[+加关注](javascript:void\(0\);)

0

0

currentDiggType = 0;

[«](https://www.cnblogs.com/Marktowin/p/19492834) 上一篇： [Mybatis-Plus更新操作时的一个坑](https://www.cnblogs.com/Marktowin/p/19492834 "发布于 2026-01-16 16:37")

posted @ 2026-01-21 15:15  [Marktowin](https://www.cnblogs.com/Marktowin)  阅读(9)  评论(0)    [收藏](javascript:void\(0\))  [举报](javascript:void\(0\))
