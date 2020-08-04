### 跨域是什么意思???

![img](https://mmbiz.qpic.cn/mmbiz_png/vwiav2ugSZtvAQbCCibLiaYVDS4zDwicpfgP6MBricF9MnfBA1hyMAdGGOia5t3eccU2qiazd3yQtBaIl2vPPf0Olgt1w/640?wx_fmt=jpeg)
做开发的小伙伴多少会看过这个图吧  下面我们就来说说跨域是什么

要了解跨域首先，我们需要了解一些前置知识:

​			一个URL是怎么组成的：

> // 协议 + 域名（子域名 + 主域名） + 端口号 + 资源地址
> http://www.baidu.com:8080/

只要**协议，子域名，主域名，端口号**这四项组成部分中有一项不同
就可以认为是不同的域，不同的域之间互相访问资源，就被称之为跨域。

随着前后端分离开发的越来越普及，会经常遇到跨域的问题，当我们在浏览器中看到这样的错误时，就需要意识到遇到了跨域：



### 如何解决跨域问题呢?

首先，我们使用`vue-cli`来快速构建一个前端项目，然后使用`axios`来向后台发送ajax请求。最后在控制台中打印出返回信息。这里就不再多做赘述，后面我会单独写一篇文章来讲一下如何使用vue-cli快速创建一个vue项目。

> 这里不再讲解使用`jsonp`的方式来解决跨域，因为`jsonp`方式只能通过`get`请求方式来传递参数，而且有一些不便之处。

下面的几种方式都是通过**跨域访问技术CORS(Cross-Origin Resource Sharing）**来解决的。具体的实现原理我们不做深入的探究，这节课的目的是解决跨域问题~

#### 方法一：注解方式

在Spring Boot 中给我们提供了一个注解`@CrossOrigin`来实现跨域，这个注解可以实现方法级别的细粒度的跨域控制。我们可以在类或者方添加该注解，如果在类上添加该注解，该类下的所有接口都可以通过跨域访问，如果在方法上添加注解，那么仅仅只限于加注解的方法可以访问 如下代码 我们在类名上加上一个注解`@CrossOrigin`来实现跨域。

```
@RestController
@RequestMapping("/user")
@CrossOrigin
public class UserController {
   @Autowired
    private UserService userService;

   @RequestMapping("/findAll")
    public Object findAll(){
        return userService.list();
    }
}
```

现在再去测试一下：

![img](https://mmbiz.qpic.cn/mmbiz_png/vwiav2ugSZtvAQbCCibLiaYVDS4zDwicpfgPqMEjXNRgUHkGnia6iaDODlaiacbiaOicicaAs65WjialIlyicbQPxXYmFWut7w/640?wx_fmt=jpeg)

测试结果为成功！

#### 方法二：通过实现WebMvcConfigurer类

这里可以通过实现`WebMvcConfigurer`接口中的`addCorsMappings()`方法来实现跨域。

```
@Configuration
class CORSConfiguration implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**");
    }
}
```

下面我们将刚刚加上的注解给注释掉，看看能不能访问到这个接口：

![img](https://mmbiz.qpic.cn/mmbiz_png/vwiav2ugSZtvAQbCCibLiaYVDS4zDwicpfgP8N5DIV1c5IyZT9xC9MIsFPS6ctiaXfCFn3SGe4icf792RyrMykqJS4hQ/640?wx_fmt=jpeg)

不出我们所料，果然还是可以的~

#### 方法三：通过Filter方式

我们可以通过实现Fiter接口在请求中添加一些Header来解决跨域的问题：

```
@Component
public class CORSFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse res = (HttpServletResponse) response;
        res.addHeader("Access-Control-Allow-Credentials", "true");
        res.addHeader("Access-Control-Allow-Origin", "*");
        res.addHeader("Access-Control-Allow-Methods", "GET, POST, DELETE, PUT");
        res.addHeader("Access-Control-Allow-Headers", "Content-Type,X-CAF-Authorization-Token,sessionToken,X-TOKEN");
        if (((HttpServletRequest) request).getMethod().equals("OPTIONS")) {
            response.getWriter().println("ok");
            return;
        }
        chain.doFilter(request, response);
    }
    @Override
    public void destroy() {
    }
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
}
```

不出意外的话，应该也可以在控制台获取到返回信息。

#### 方法四：单个接口添加response header

方法同上  实现方式的不同点在于 只在单个接口中的response中添加header 使得单个接口能够跨域访问


#### Nginx配置跨域

如果我们在项目中使用了Nginx，可以在Nginx中添加以下的配置来解决跨域
（原理其实和Filter类似，只不过把活儿丢给了运维🤔）

```
location / {
   add_header Access-Control-Allow-Origin *;
   add_header Access-Control-Allow-Headers X-Requested-With;
   add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS;

   if ($request_method = 'OPTIONS') {
     return 204;
   }
}
```