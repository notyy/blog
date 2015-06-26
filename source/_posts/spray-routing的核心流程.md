title: spray-routing的核心流程
date: 2015-06-26 17:17:43
tags:
- spray
- akka
- restful
- scala
categories:
- 技术
---
最近我们在一个项目上使用[spray](http://spray.io/)来发布restful service。
![spray logo](http://7u2h31.com1.z0.glb.clouddn.com/spray_logo_c.png)
spray是个性能很好而且功能非常完整的service框架，包含很多组件，从底层http服务器到高层的rest路由DSL都有。一般简单的应用就使用和掌握好最高层的spray-routing就够用。本文主要讲spray-routing，不及其余。 spray整体的设计理念，spray和akka的关系留待以后的博客。
spray-routing上手很容易，但是有一些比较独特的概念和设计。如果没有一定的理解，就会发现当系统复杂到一定程度时对于有些需求不知道该怎么实现了。
所以写这篇博客先解释一下[spray的核心流程](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/)，之后会再写文章讲最核心的Directive（指令），方便大家掌握使用。
spray发布http service的流程如下：
![Image Title](http://7u2h31.com1.z0.glb.clouddn.com/spray流程概念1.png)
整个流程由spray框架控制，http连接处理由spray-can或spray-servlet负责，大部分情况下，开发人员只要定义路由——url和业务服务的映射——以及对应的业务服务即可，注意这个路由定义并不是一个配置文件，而是spray-routing定义的一套scala的DSL。
请求到达时，spray会先查找路由定义，如果请求的URL没有找到最终能完成请求的服务则会拒绝(reject)。 如果找到，则spray会根据你在路由定义里的配置，把请求参数转成业务对象（比如用json4s把json请求转换成scala对象，需要用Entity指令来定义），然后调用业务服务。调用可能有三种结果：
- 业务处理正常返回，则将返回的业务对象根据配置的转换方式转换回HttpResponse，再返回给客户端
- 调用业务服务超时，则交由一个可覆盖的超时处理器处理，默认实现是返回500内部服务器错，并带一个默认的错误信息
```scala
def timeoutRoute: Route = complete(
  InternalServerError,
  "The server was not able to produce a timely response to your request.")
```
- 业务服务抛异常，跟超时处理一样会被交给一个可自定义的异常处理块去统一处理

我们的路由服务一般继承HttpService，HttpService继承自HttpServiceBase，其中提供了runRoute方法，也就是路由的入口。
```scala
def runRoute(route: Route)(implicit eh: ExceptionHandler, rh: RejectionHandler, ac: ActorContext,rs: RoutingSettings, log: LoggingContext): Actor.Receive
```
runRoute的参数是Route:
```scala
type Route = RequestContext ⇒ Unit
```
是RequestContext => Unit的类型别名
我们按照spray例子写的path("xx")...其实就是定义了一个`RequestContext => Unit`的函数，也就是如何从请求上下文里解析请求内容，调用业务服务。 比较奇怪的是返回类型是Unit，spray会调用RequestContext里包含的responder成员来负责将响应返回给客户端。 据spray-routing文档里说是为了”non-blocking"和"actor friendly"，但实际上在spray的后续版本，也就是akka-http里把这个返回类型改成了RouteResult,也就是`RequestContext => RouteResult`，这样感觉更合理,更对应于`HttpRequest => HttpResponse`的本质[^1]。
我们完全可以定义一个`RequestContext ⇒ Unit`类型的路由，然后自己从RequestContext里解析出请求数据，自己做数据转换，自己决定应该调用什么服务（实际上有些时候我们确实要这么做）。 但大部分时候我们可以用spray-routing通过一组`Directive`——翻译成中文就是**指令**——提供的路由DSL来定义我们的路由。这也是spray-routing提供的最核心的功能。
比如官方文档里的路由例子：
```scala
import spray.routing._
import Directives._

val route: Route =
  path("order" / IntNumber) { id =>
    get {
      complete {
        "Received GET request for order " + id
      }
    } ~
    put {
      complete {
        "Received PUT request for order " + id
      }
    }
  }
```
path、get、put、complete都是`指令`，spray的指令实现用了一种比较复杂的模式叫做[磁铁模式(maget pattern)](http://spray.io/blog/2012-12-13-the-magnet-pattern/)。我以后另外写文介绍，本文只讲理念。
拿上面代码里的path为例，directive的一般形式为：
```scala
name(arguments) { extractions =>
  ... // inner Route
}
```
spray对`RequestContext => Unit`要完成的事情通过directive做了很好的抽象：
1. 转换——将RequestContext做一些转换再传给下一级路由
2. 过滤——拒绝不符合条件的请求
3. 抽取——从RequestContext里抽取一些信息，使之在下级路由中可用，比如上例中的`id =>`
4. 完成请求——比如上例中的`complete{ }`

对于过滤功能而言，还需要能“并联”——如果这个路径与请求不匹配，spray要去尝试下一个路径，有点像嵌套的模式匹配。 在spray-routing里并联用的是操作符 “~” 在前例中的get和put分支的并联可以看得很清楚。 但”~“**不是唯一**的把directive组合起来的方法，当路由定义变得庞大的时候，我们会需要某种方法把大量类似的结构抽取出来免得写出一棵巨大无比的路由树，我将在另一篇文章中对此进行介绍。
再回头看一下前面的流程图，除了正常路由、正常处理外还有*拒绝*，*异常*，*超时*三个分支。看上去好像我们只定义了正常处理的逻辑，实际上是我们的spray路由的入口`runRoute`这个方法偷偷做了默认处理
```scala
def runRoute(route: Route)(implicit eh: ExceptionHandler, rh: RejectionHandler, ac: ActorContext,rs: RoutingSettings, log: LoggingContext): Actor.Receive
```
里面有隐式参数ExceptionHandler和RejectionHandler,以及一个隐藏的超时处理：
```scala
case Timedout(request: HttpRequest) ⇒ runRoute(timeoutRoute)(eh, rh, ac, rs, log)(request)
```
默认的拒绝实现对于常见的拒绝原因都给出正确的错误码和不错的返回信息，比如
```scala
case AuthorizationFailedRejection :: _ ⇒
      complete(Forbidden, "The supplied authentication is not authorized to access this resource")
```

异常处理器和超时处理器也一样，如果你需要定制也可以定制自己的处理器，方法见
[Reject Handler](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/rejections/#rejectionhandler)
[Exception Handler](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/exception-handling/)
[Timeout Handler](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/timeout-handling/)

[^1]: [akka-streams-0.7-released](http://akka.io/news/2014/09/12/akka-streams-0.7-released.html)
