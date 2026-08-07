# Spring MVC

---

## 一、Spring MVC 执行流程 🔥

```
用户请求 → DispatcherServlet → HandlerMapping → HandlerAdapter → Handler(Controller) → 返回 ModelAndView → ViewResolver → 视图渲染 → 响应
```

### 详细流程

```
1. 请求到达 DispatcherServlet（前端控制器）
2. DispatcherServlet 调用 HandlerMapping 查找 Handler（找到 Controller）
3. HandlerMapping 返回 HandlerExecutionChain（Handler + Interceptors）
4. DispatcherServlet 调用 HandlerAdapter 执行 Handler
5. HandlerAdapter 调用具体的 Controller 方法
6. Controller 返回 ModelAndView
7. HandlerAdapter 将 ModelAndView 返回给 DispatcherServlet
8. DispatcherServlet 调用 ViewResolver 解析视图
9. ViewResolver 返回 View 对象
10. View 渲染并返回响应
```

---

## 二、核心组件

| 组件 | 作用 |
|------|------|
| DispatcherServlet | 前端控制器，统一分发请求 |
| HandlerMapping | 映射请求到 Handler（@RequestMapping → HandlerMethod） |
| HandlerAdapter | 适配器，用统一的接口执行不同的 Handler |
| HandlerInterceptor | 拦截器（preHandle → Controller → postHandle → afterCompletion） |
| ViewResolver | 视图解析器（JSP、Thymeleaf 等） |
| HttpMessageConverter | 消息转换器（JSON ↔ 对象） |

---

## 三、拦截器 vs 过滤器

| 对比 | 过滤器（Filter） | 拦截器（Interceptor） |
|------|-----------------|---------------------|
| 所属 | Servlet 容器 | Spring MVC |
| 作用范围 | 所有请求 | 只拦截 Controller 请求 |
| 能力 | 只能操作 request/response | 可以访问 Handler 和 ModelAndView |
| 执行顺序 | Filter → Interceptor → Controller |

---

## 四、RESTful API

```java
@RestController  // = @Controller + @ResponseBody
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User get(@PathVariable Long id) { }

    @PostMapping
    public User create(@RequestBody @Valid User user) { }

    @PutMapping("/{id}")
    public User update(@PathVariable Long id, @RequestBody User user) { }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) { }
}
```

---

## 五、参数绑定与校验

```java
@PostMapping
public Result create(@RequestBody @Validated User user, BindingResult result) {
    if (result.hasErrors()) {
        return Result.fail(result.getAllErrors());
    }
}

public class User {
    @NotBlank(message = "姓名不能为空")
    private String name;

    @Min(value = 1, message = "年龄必须大于0")
    @Max(value = 150, message = "年龄不能超过150")
    private int age;

    @Email(message = "邮箱格式不正确")
    private String email;
}
```

---

## 六、统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleValidation(MethodArgumentNotValidException e) {
        return Result.fail(e.getBindingResult().getFieldError().getDefaultMessage());
    }

    @ExceptionHandler(BusinessException.class)
    public Result handleBusiness(BusinessException e) {
        return Result.fail(e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result handleUnknown(Exception e) {
        log.error("未知异常", e);
        return Result.fail("系统繁忙");
    }
}
```

---

## 面试追问集

**Q：@RequestMapping 和 @GetMapping 有什么区别？**

@GetMapping 是 @RequestMapping(method = RequestMethod.GET) 的简写，另有 @PostMapping、@PutMapping、@DeleteMapping、@PatchMapping。

**Q：@PathVariable 和 @RequestParam 的区别？**

- @PathVariable：从 URL 路径获取参数（/users/{id}）
- @RequestParam：从 URL 查询参数获取（/users?id=1）

**Q：Spring MVC 如何解决 GET 和 POST 中文乱码？**

- GET：Tomcat 的 server.xml 中 URIEncoding="UTF-8"
- POST：web.xml 配置 CharacterEncodingFilter；Spring Boot 中 `server.servlet.encoding.charset=UTF-8` 即可
