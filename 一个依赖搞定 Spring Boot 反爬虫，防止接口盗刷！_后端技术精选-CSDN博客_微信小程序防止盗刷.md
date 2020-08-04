# 一个依赖搞定 Spring Boot 反爬虫，防止接口盗刷！_后端技术精选-CSDN博客_微信小程序防止盗刷

> 作者：小哈学Java来源：http://suo.im/5tZIlqkk-anti-reptile 是适用于基于 spring-boot 开发的分布式系统的反爬虫组件。系统要求基于 spr..._微信小程序防止盗刷

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9KZlRQaWFoVEhKaG9rdk96RjNQbEhGczlpYlRDcEtGTDhOaHo5Z1c2VmljN2NuaWI4MlhZYWJvMzB4RGRrYzJXU0ZwVHFvcTZHb201SGliMEFTZmJLSmVEUW1BLzY0MA?x-oss-process=image/format,png)

作者：小哈学Java

来源：http://suo.im/5tZIlq

kk-anti-reptile 是适用于基于 spring-boot 开发的分布式系统的反爬虫组件。  

系统要求
----

*   基于 spring-boot 开发(spring-boot1.x, spring-boot2.x均可)
    
*   需要使用 redis
    

工作流程
----

kk-anti-reptile 使用基于 Servlet 规范的的 Filter 对请求进行过滤，在其内部通过 spring-boot 的扩展点机制，实例化一个 Filter，并注入到 Spring 容器 FilterRegistrationBean 中，通过 Spring 注入到 Servlet 容器中，从而实现对请求的过滤。

在 kk-anti-reptile 的过滤 Filter 内部，又通过责任链模式，将各种不同的过滤规则织入，并提供抽象接口，可由调用方进行规则扩展。

Filter 调用则链进行请求过滤，如过滤不通过，则拦截请求，返回状态码 509，并输出验证码输入页面，输出验证码正确后，调用过滤规则链对规则进行重置。

目前规则链中有如下两个规则

### ip-rule

ip-rule 通过时间窗口统计当前时间窗口内请求数，小于规定的最大请求数则可通过，否则不通过。时间窗口、最大请求数、ip 白名单等均可配置。

### ua-rule

ua-rule 通过判断请求携带的 User-Agent，得到操作系统、设备信息、浏览器信息等，可配置各种维度对请求进行过滤。

命中规则后
-----

命中爬虫和防盗刷规则后，会阻断请求，并生成接除阻断的验证码，验证码有多种组合方式，如果客户端可以正确输入验证码，则可以继续访问

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9HdnRER0tLNHVZa2liaWNPN1Q3a250cTRkUWdEZ3RRWVBHZHNzeWNRekxwUGhvR3M1NTJFWDB4TVVDcVZ1YWFWWHZ1NUlmUkxNd3N3ZEN5d1lxc1Q2REdRLzY0MA?x-oss-process=image/format,png)

验证码有中文、英文字母+数字、简单算术三种形式，每种形式又有静态图片和 GIF 动图两种图片格式，即目前共有如下六种，所有类型的验证码会随机出现，目前技术手段识别难度极高，可有效阻止防止爬虫大规模爬取数据

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2dpZi9HdnRER0tLNHVZa2liaWNPN1Q3a250cTRkUWdEZ3RRWVBHRERUT1hTTVkzNHZwUGNPd1lLUnNRQjVpYzA0cmppYThORVZ2QWt3VG5hQXNvV2NzQlFFc0l1WkEvNjQw?x-oss-process=image/format,png)

接入使用
----

后端接入非常简单，只需要引用 kk-anti-reptile 的 maven 依赖，并配置启用 kk-anti-reptile 即可加入 maven 依赖

<dependency\>  
    <groupId\>cn.keking.projectgroupId\>  
    <artifactId\>kk-anti-reptileartifactId\>  
    <version\>1.0.0-SNAPSHOTversion\>  
dependency\>  

配置启用 kk-anti-reptile

anti.reptile.manager.enabled=true  

前端需要在统一发送请求的 ajax 处加入拦截，拦截到请求返回状态码 509 后弹出一个新页面，并把响应内容转出到页面中，然后向页面中传入后端接口 baseUrl 参数即可，以使用 axios 请求为例：

import axios from 'axios';  
import {baseUrl} from './config';

axios.interceptors.response.use(  
  data => {  
    return data;  
  },  
  error => {  
    if (error.response.status === 509) {  
      let html = error.response.data;  
      let verifyWindow = window.open("","\_blank","height=400,width=560");  
      verifyWindow.document.write(html);  
      verifyWindow.document.getElementById("baseUrl").value = baseUrl;  
    }  
  }  
);  
export default axios;

注意
--

*   apollo-client 需启用 bootstrap
    

使用 apollo 配置中心的用户，由于组件内部用到 @ConditionalOnProperty，要在 application.properties/bootstrap.properties 中加入如下样例配置，(apollo-client 需要 0.10.0 及以上版本）详见 apollo bootstrap 说明

apollo.bootstrap.enabled = true  

*   需要有 Redisson
    

连接如果项目中有用到 Redisson，kk-anti-reptile 会自动获取 RedissonClient 实例对象; 如果没用到，需要在配置文件加入如下 Redisson 连接相关配置:

spring.redisson.address=redis://192.168.1.204:6379  
spring.redisson.password=xxx  

配置一览表
-----

在 spring-boot 中，所有配置在配置文件都会有自动提示和说明，如下图：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9HdnRER0tLNHVZa2liaWNPN1Q3a250cTRkUWdEZ3RRWVBHdGJvNkNaaWFsR3BqRjFBenJ3VUx3WGRqVm5PYUZGaDZEaWFidnBUZEFQYzBjRUhUUVE0SzJlWkEvNjQw?x-oss-process=image/format,png)

所有配置都以 anti.reptile.manager 为前缀，如下为所有配置项及说明:

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9HdnRER0tLNHVZa2liaWNPN1Q3a250cTRkUWdEZ3RRWVBHbGdWWDN4OXJmbk9pYzNRZ2VBbHNjUk9iNTZOR1JYTUdCdEdzWXRNeEdjRWd4UTVSSXhTUUZGdy82NDA?x-oss-process=image/format,png)
