## 简介

Dubbo Spring Cloud 基于 Dubbo Spring Boot 2.7.1[1] 和 Spring Cloud 2.x 开发，无论开发人员是 Dubbo 用户还是 Spring Cloud 用户，
都能轻松地驾驭，并以接近“零”成本的代价使应用向上迁移。Dubbo Spring Cloud 致力于简化 Cloud Native 开发成本，提高研发效能以及提升应用性能等目的。

Dubbo Spring Cloud 首个 Preview Release，随同 Spring Cloud Alibaba `0.2.2.RELEASE` 和  `0.9.0.RELEASE` 一同发布[2]，
分别对应 Spring Cloud Finchley[3] 与 Greenwich[4] (下文分别简称为 “F” 版 和 “G” 版) 。

目前 Dubbo Spring Cloud 仍处于 preview 阶段，请等待成熟后再应用于生产环境。




## 功能

由于 Dubbo Spring Cloud 构建在原生的 Spring Cloud 之上，其服务治理方面的能力可认为是 Spring Cloud Plus，
不仅完全覆盖 Spring Cloud 原生特性[5]，而且提供更为稳定和成熟的实现，特性比对如下表所示：

| 功能组件                                             | Spring Cloud                           | Dubbo Spring Cloud                                     |
| ---------------------------------------------------- | -------------------------------------- | ------------------------------------------------------ |
| 分布式配置（Distributed configuration）              | Git、Zookeeper、Consul、JDBC           | Spring Cloud 分布式配置 + Dubbo 配置中心[6]          |
| 服务注册与发现（Service registration and discovery） | Eureka、Zookeeper、Consul              | Spring Cloud 原生注册中心[7] + Dubbo 原生注册中心[8] |
| 负载均衡（Load balancing）                           | Ribbon（随机、轮询等算法）             | Dubbo 内建实现（随机、轮询等算法 + 权重等特性）        |
| 服务熔断（Circuit Breakers）                         | Spring Cloud Hystrix                   | Spring Cloud Hystrix + Alibaba Sentinel[9] 等        |
| 服务调用（Service-to-service calls）                 | Open Feign、`RestTemplate`             | Spring Cloud 服务调用 + Dubbo `@Reference`             |
| 链路跟踪（Tracing）                                  | Spring Cloud Sleuth[10] + Zipkin[11] | Zipkin、opentracing 等                                 |


## [示例代码](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-examples/spring-cloud-alibaba-dubbo-examples/README_CN.md)





















---

[1]: 从 2.7.0 开始，Dubbo Spring Boot 与 Dubbo 在版本上保持一致

[2]: Preview releases of Spring Cloud Alibaba are available: 0.9.0, 0.2.2, and 0.1.2 - <https://spring.io/blog/2011/04/11/preview-releases-of-spring-cloud-alibaba-are-available-0-9-0-0-2-2-and-0-1-2>

[3]: 目前最新的 Spring Cloud “F” 版的版本为：`Finchley.SR2` - <https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html>

[4]: 当前Spring Cloud “G” 版为 `Greenwich.RELEASE`

[5]:  Spring Cloud 特性列表 - <https://cloud.spring.io/spring-cloud-static/Greenwich.RELEASE/single/spring-cloud.html#_features>

[6]:  Dubbo 2.7 开始支持配置中心，可自定义适配 - <http://dubbo.apache.org/zh-cn/docs/user/configuration/config-center.html>

[7]: Spring Cloud 原生注册中心，除 Eureka、Zookeeper、Consul 之外，还包括 Spring Cloud Alibaba 中的 Nacos

[8]: Dubbo 原生注册中心 - <http://dubbo.apache.org/zh-cn/docs/user/references/registry/introduction.html>

[9]: Alibaba Sentinel：Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性 - <https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D>，目前 Sentinel 已被 Spring Cloud 项目纳为 Circuit Breaker  的候选实现 - <https://spring.io/blog/2011/04/8/introducing-spring-cloud-circuit-breaker>

[10]:Spring Cloud Sleuth - <https://spring.io/projects/spring-cloud-sleuth>

[11]: Zipkin - <https://github.com/apache/incubator-zipkin>