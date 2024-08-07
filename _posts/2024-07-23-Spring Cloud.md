# Spring Cloud


备注：dubbo 只是一个远程调用框架，而且后面都停更了，而 Spring Cloud 是一个完整的解决方案，只要用到微服务，就不能只有一个远程调用；虽然它的远程调用效率高，但是一般的服务调用就搞个 http（feign） 就行了，也不要求一定要很高的效率，就像 go 并发性能高，但是用的人少（先来后到，生态问题）。

## 微服务和单机的优缺点

单机优点：
如果业务不复杂，开发很简单，便于小型团队管理；
部署简单。
缺点：
随着业务变得复杂，项目变得庞大比较难维护；
重新部署影响整个系统。

微服务优点：
开发效率高，团队可以并行开发不同的服务；
各个服务独立开发、部署和扩展，灵活性高；
单个服务单点故障不会影响整个系统，高可用。
缺点：
确保服务之间的数据一致性变得困难；
部署变的复杂；
会衍生出分布式问题，如分布式事务、分布式 ID等。

## 微服务和分布式的区别

微服务：一个服务拆分成多个服务，部署在不同机器上。
分布式：并不局限于服务，比如分布式 id、分布式事务。

## 注册中心

管理服务，起一个信息交换的作用，服务把信息（ip 地址、端口号）注册上去，就能被其他服务知道，进行远程调用。

注：注册中心挂掉了，还是能远程调用，因为 consumer 会把 provider 信息拉到本地缓存（不用每次都去注册中心拉取一遍，rpc 调用不会翻倍），除非信息变了。

### eureka

诟病：
基于 AP 思想，数据容易不同步，延时比较高；
后面停更了；
三级缓存、默认配置。

### nacos

支持 AP 和 CP 思想；
是阿里巴巴的，生态好。

### consul

## 配置中心

## 网关

权限校验；
限流；
路由分发；
负载均衡（服务端的负载均衡）（nginx）。
负载均衡算法：随机、轮询、权重、哈希

### zuul

### gateway

## 负载均衡

客户端（服务内部之间相互调用）的负责均衡

### ribbon

## 限流

令牌桶算法；
滑动窗口算法。
限流桶：削峰
漏桶：直接拿

### sentinel

## 远程调用

### restTemplate

ip + 端口号 + 资源路径

缺点：ip 地址变了，要改配置、改代码。

### OpenFeign

服务名 + 资源路径

把 feign 接口定义后，使用起来就像直接调自己本地的 service。

## 熔断、降级、监控

如果某个 provider 服务不可用或者响应时间太长时，就会进行降级（备用方法），进而熔断该服务的调用，快速响应错误；
如果某个 consumer 服务访问 provider 服务得不到响应，就会降级，执行预先设置好的一个解决方案。
参考资料：众筹网

### hystrix

