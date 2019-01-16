Spring Cloud 官方宣布 Spring Cloud Netflix 进入维护状态，后续不再进行更新已成为事实。作为开发者的我们，如何使用极简的方式替换现有的 Netflix 组件成为了首要解决的问题。Spring Cloud Alibaba 实现了 Spring Cloud 服务注册的标准规范，这就天然的给开发者提供了一种非常便利的方式将服务注册中心的 Eureka 迁移到商业化的组件 ANS 。 

下面是大致整理了 SCN 迁移 SCA 之 Eureka 迁移的三种方案。

* 方案 1: 重启进程迁移

   客户改造完代码后，直接停掉所有的进程并重新部署新代码到 EDAS 上。

	缺点：需要停机

* 方案 2: 切流迁移

	即准备两套环境，一套是迁移后的环境，一套现有的环境。总的流量入口控制访问流量流入迁移后的环境还是现有环境，两环境之间的服务是不可调用的。当迁移后的环境跑一段时间服务运行稳定后，对现有的环境机器总体停机，流量全部打到迁移后的环境；如果有问题，入口流量切断迁移后的环境，流量又访问现有的环境。
	
	缺点：维护机器成本增加一倍


* 方案 3: 混存迁移

	在现有的环境里，对部分服务进行迁移的改造，部署后，服务既注册于 Eureka,又注册于 ANS 。当所有的服务都注册于 ANS 时，迁移完成，直接将 Eureka 的服务下线调。
	
	缺点：实现起来较有挑战，需要双注册，双定阅。
	

接下来将重点讨论下 **混存迁移** 的具体实现。	
	
混存迁移的核心思路是在 ANS Starter 中，支持多注册中心的双注册/双订阅，以此来达到不需要全部重启进程，也不需要增加一套环境的目的，降低成本，提高服务的在线服务时长。
	
在 Spring Cloud 服务注册与发现的规范中，只能支持单注册中心，这主要是因为在 Spring Cloud 的服务注册与发现过程中，限制了在 Spring 的 Ioc 容器中只能存在一个 AnsServiceRegistry、AnsRegistration、AbstractAutoServiceRegistration 三种类型的 Bean。**因此首要解决的第一个问题是：如何解决 Eureka Starter 和 ANS Starter 共存的问题 ？**

## 如何解决 Eureka 和 ANS Starter 可同时引入

导致这个问题的根源是因为在 Spring Cloud 的服务注册与发现过程中，限制了在 Spring 的 Ioc 容器中只能存在一个 AnsServiceRegistry、AnsRegistration、AbstractAutoServiceRegistration 三种类型的 Bean。因此只需要当两者同时引入时，ANS 的 AnsServiceRegistry、AnsRegistration、AbstractAutoServiceRegistration 三种类型的 Bean 不通过 Spring 来初始化即可。因此问题也就转化为 **如何判断当前应用中同时引入了 Eureka 和 ANS Starter，从而来控制是否需要通过 Spring 的方式来初始化和服务注册与发现相关的那个三个 Bean 。** 

每一个实现 Spring Cloud 服务注册与发现的标准规范中，都需要实现 **AbstractAutoServiceRegistration** 这个抽象类，Eureka 也不例外。在 Eureka Starter 中，这个类的实现类是：**org.springframework.cloud.netflix.eureka.serviceregistry.EurekaAutoServiceRegistration** 。因此就可以根据当前 JVM 进程中是否加载了这个 Class 为前提条件，来控制和服务注册与发现相关 Bean 的初始化工作。因此基于 Spring 的 Condition 扩展了两个和混存迁移相关的 Condition，分别是：

* MigrateOnConditionClass： 混存迁移打开的条件。主要就是判断当前 JVM 进程中是否加载了 **org.springframework.cloud.netflix.eureka.serviceregistry.EurekaAutoServiceRegistration** 这个类。 从而手动实例化和服务注册与发现相关的那三个类的实例。


* MigrateOnConditionMissingClass：混存迁移关闭的条件。主要就是判断当前 JVM 进程中是否加载了 **org.springframework.cloud.netflix.eureka.serviceregistry.EurekaAutoServiceRegistration** 这个类。 没有加载的话通过 Spring Boot 的 Java 配置的方式来实例化和服务注册与发现相关的那三个 Bean。

**说明：** 当前实现不仅仅是判断是否加载了和 Eureka 相关的 Class，和 Consul 相关的 Class（**org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistration**） 也做了判断。因此如果你的服务注册中心使用的是 Consul，也是可以支持平滑的从 Consul 迁移到 ANS。


## 如何解决双注册

解决了 Eureka 和 ANS Starter 共存的问题，接下来要解决的另外一个核心问题就是：**如何实现双注册 ？** 

在 **MigrateOnConditionClass** 条件满足的情况下，会通过 Java 配置的方式来初始化一个和服务注册相关的 Bean，即 **MigrateServiceRegistry**。这个 Bean 会对 **WebServerInitializedEvent** 这个事件做监听，从而在整个应用启动的这个阶段(Web Server 已经初始化完成)，将服务注册到 ANS。


## 如何解决双订阅

如何解决双订阅，核心都聚焦在 **ServerList** 这个接口的实现类上。简单来说，Spring Cloud 会为当前这个服务依赖的每一个后端服务都会初始化一个 Spring Context，初始化这个上下文的时候，会初始化实现 **ServerList** 这个接口的 Bean。ServerList 这个接口定义了可获取某个服务的服务实例列表的能力。因此在实现混存迁移，如何解决双订阅的切入点就是可以拦截到 ServerList 实现类的行为，比如 **getInitialListOfServers** 或者 **getUpdatedListOfServers**，从而才可以 merge 从 ANS 获取服务的服务列表，然后做一个取舍。原理图如下所示：

![双订阅](http://edas.oss-cn-hangzhou.aliyuncs.com/deshao/pictures/server_list.png)


因此在实现双订阅的时候，会给每个后端服务的 Spring Context 初始化一个 **BeanPostProcessor** 类型的 Bean，其目的就是将会给 ServerList 类型的 Bean 返回一个被代理后的 Bean，从而有能力拦截到 ServerList 实现类的行为，主要就是 **getInitialListOfServers** 、 **getUpdatedListOfServers** 这两个行为。当这两个行为被拦截到之后，会从 Eureka 和 ANS 两个注册中心获取当前服务的实例列表。如果一个实例两个注册中心都有，将会用来自于 ANS 的实例替换来自于 Eureka 的实例，从而达到混存平滑迁移的效果。当一个服务的服务实例列表都被来自于 ANS 所替代时，也说明当前这个服务实例已经完成了混存平滑迁移。如何来查看当前某个服务的服务列表状态，可通过 id 为 **migrate** 的一个 Endpoint 查询，同时 Endpoint 中也展示了某一个服务实例被调用的次数，可以清晰的看到被迁移后的服务实例发生服务调用时的一个次数情况。

## 如何使用


当您的注册中心需要从 Eureka 或者 Consul 迁移到 ANS 时，整过迁移的过程将要经历那几个阶段呢？


### 迁移中

在这个阶段，你的应用中将会同时引入 Eureka 和 ANS Starter。如下所示：

```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alicloud-ans</artifactId>
        <version>0.2.2.BUILD-SNAPSHOT</version>
    </dependency>
	<!-- 其他依赖 -->
</dependencies>

```
**注意：** 版本号是当前这个阶段的一个版本号，并且是 0.2.2 版本及以上。真实实践起来依赖的版本号以官网为主。

迁移过程中如果发现服务调用失败率较高，怀疑混存迁移这个实现方案有问题，可以在运行态来关闭它并及时向我们反馈。关闭的前提是你的 Spring Cloud 应用中使用了 Spring Cloud Config 的这个特性。即支持 Spring Cloud Config 的外部化配置(不局限于某一个配置存储源)，当监听到 **spring.cloud.alicloud.migrate.ans.switch** 这个配置设置为 **false** 时，这个时候将会关闭混存迁移。

运行态关闭混存平滑迁移内部的实现原理是实现了一个 **ApplicationListener**，并对 **RefreshEvent** 这个事件做了处理，**RefreshEvent** 这个事件是当 Spring Cloud Config 的外部化配置有变更时，会发布这么一个事件，从而感知到外部化配置发生了变化，从 Spring 的 Enviroment 中获取当前 **spring.cloud.alicloud.migrate.ans.switch** 这个配置值。


在迁移中，可以访问 Endpoint id 为 **migrate**，来查看当前迁移的一个基本状态，以下是在Spring Boot 2.x 版本下访问的显示结果。


```json
Endpoint Url: http://localhost:port/actuator/migrate

{
  "sc-migrate-eureka-consumer": {
    "30.5.125.22:40051": {
      "server": {
        // **** 省略
        "instanceInfo": {
          "instanceId": "30.5.125.22:sc-migrate-eureka-consumer:40051",
          "app": "SC-MIGRATE-EUREKA-CONSUMER",
          "appGroupName": null,
          "ipAddr": "30.5.125.22",
          "sid": "na",
          "homePageUrl": "http://30.5.125.22:40051/",
          "statusPageUrl": "http://30.5.125.22:40051/actuator/info",
          "healthCheckUrl": "http://30.5.125.22:40051/actuator/health",
          "secureHealthCheckUrl": null,
          "vipAddress": "sc-migrate-eureka-consumer",
          "secureVipAddress": "sc-migrate-eureka-consumer",
          "countryId": 1,
          "dataCenterInfo": {
            "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
            "name": "MyOwn"
          },
          // ****
      },
      "callCount": 8 //服务调用时选择该实例被调用的次数
    },
    "30.5.125.22:40052": {
      "server": {
        // **** 省略
        "metaInfo": {
          "instanceId": "30.5.125.22:sc-migrate-eureka-consumer:40052",
          "appName": "sc-migrate-eureka-consumer",
          "serverGroup": null,
          "serviceIdForDiscovery": "sc-migrate-eureka-consumer"
        },
        "metadata": {
          "source": "ANS" // 如果该实例是来自于 ANS，将会有此标识
        }, 
      	// **** 省略
       },
      "callCount": 9 //服务调用时选择该实例被调用的次数
    }
  }
}

```


### 迁移后

运行一段时间后，如果发现服务之间调用稳定，这个时候就需要将 Eureka 的依赖给去掉。去掉的方式可以有两种选择：

* 一步到位将所有应用对 Eureka 依赖给去掉。

	这种方式的 **优点** 是工作量批次不需要分多次，一步到位去掉即可。但是 **缺点** 也较明显，如果去掉后发现有问题，需要回滚，那这个时候是应用全量的回滚，将会影响服务的在线可服务时长。

* 逐个的将应用对 Eureka 依赖给去掉。

	这种方式相比于前一种，优缺点刚好反过来。即 **优点** 是如果去掉后发现有问题，可以及时的对某个服务实例进行回滚。**缺点** 是更改的工作量批次可能会高。

迁移后，访问 Endpoint id 为 **migrate** 的结果如下所示：


```json
{
  "sc-migrate-eureka-consumer": {
    "30.5.125.22:40051": {
      "server": {
        // **** 
        "metaInfo": {
          "instanceId": "30.5.125.22:sc-migrate-eureka-consumer:40051",
          "appName": "sc-migrate-eureka-consumer",
          "serverGroup": null,
          "serviceIdForDiscovery": "sc-migrate-eureka-consumer"
        },
        "metadata": {
          "source": "ANS" // 实例来自于 ANS
        },
        "alive": true,
        // **** 
      },
      "callCount": 13 // 被调用次数
    },
    "30.5.125.22:40052": {
      "server": {
        // ****
        "metaInfo": {
          "instanceId": "30.5.125.22:sc-migrate-eureka-consumer:40052",
          "appName": "sc-migrate-eureka-consumer",
          "serverGroup": null,
          "serviceIdForDiscovery": "sc-migrate-eureka-consumer"
        },
        "metadata": {
          "source": "ANS" // 实例来自于 ANS
        },
        "alive": true,
        // ****
      },
      "callCount": 15 // 被调用次数
    }
  }
}
```

可以看到 **sc-migrate-eureka-consumer** 这个服务的两个实例拉取都来源于 ANS 。

## 结束语
	
整个混存的迁移过程中，我们是建议从最底层的服务依赖开始做起，依次到最上层的服务，如下所示的一个服务调用关系:

![服务调用](http://edas.oss-cn-hangzhou.aliyuncs.com/deshao/pictures/call_service.png)

那在混存迁移过程中，如上图所示最好是先从 D 服务开始，依次到 Api-Gateway，最后迁移完毕，就可以将 Eureka 的服务实例给下线了。		