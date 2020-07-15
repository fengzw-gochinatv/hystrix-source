

  

   在微服务架构中，根据业务来拆分成一个个的服务，服务与服务之间可以相互调用（RPC），在Spring Cloud可以用RestTemplate+Ribbon和Feign来调用。为了保证其高可用，单个服务通常会集群部署。由于网络原因或者自身的原因，服务并不能保证100%可用，如果单个服务出现问题，调用这个服务就会出现线程阻塞，此时若有大量的请求涌入，Servlet容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的“雪崩”效应。 

![图片](https://uploader.shimo.im/f/VyracLyHwSg55taY.png!thumbnail)

         为了解决这个问题，业界提出了断路器模型。 

  在生活中，如果电路的负载过高，保险箱会自动跳闸，以保护家里的各种电器，这就是熔断器的一个活生生例子。在Hystrix中也存在这样一个熔断器，当所依赖的服务不稳定时，能够自动熔断，并提供有损服务，保护服务的稳定性。在运行过程中，Hystrix会根据接口的执行状态（成功、失败、超时和拒绝），收集并统计这些数据，根据这些信息来实时决策是否进行熔断。

# 一、断路器简介 

>Netflix has created a library called Hystrix that implements the circuit breaker pattern. In a microservice architecture it is common to have multiple layers of service calls. 
>. —-摘自官网 

Netflix开源了Hystrix组件，实现了断路器模式，SpringCloud对这一组件进行了整合。 在微服务架构中，一个请求需要调用多个服务是非常常见的，如下图： 

![图片](https://uploader.shimo.im/f/wCcYgWgAbxEye2Gr.png!thumbnail)

较底层的服务如果出现故障，会导致连锁故障。当对特定的服务的调用的不可用达到一个阀值（Hystric 是5秒20次） 断路器将会被打开。 

![图片](https://uploader.shimo.im/f/dbxHrFP6efk4WHu3.png!thumbnail)

断路打开后，可用避免连锁故障，fallback方法可以直接返回一个固定值。 

# 二、demo演示

## **1：在ribbon使用断路器**

改造serice-ribbon 工程的代码，首先在pox.xml文件中加入spring-cloud-starter-hystrix的起步依赖： 

<dependency>

   <groupId>org.springframework.cloud</groupId>

   <artifactId>spring-cloud-starter-hystrix</artifactId>

</dependency>

* 1234

在程序的启动类SpringCloudServiceRibbonApplication 加@EnableHystrix注解开启Hystrix： 

@SpringBootApplication

@EnableDiscoveryClient

@EnableHystrix

@EnableHystrixDashboard

public class SpringCloudServiceRibbonApplication {

   public static void main(String[] args) {

      SpringApplication.*run*(SpringCloudServiceRibbonApplication.class, args);

   }

  

   @Bean

   @LoadBalanced

   RestTemplate restTemplate(){

      return new RestTemplate();

   }

}

改造UserService类，在query方法上加上@HystrixCommand注解。该注解对该方法创建了熔断器的功能，并指定了fallbackMethod熔断方法，熔断方法直接返回了一个对象，代码如下： 

@Service

public class UserService {

    @Autowired

    RestTemplate restTemplate;

@HystrixCommand(commandKey="queryCommandKey",groupKey = "queryGroup",

        threadPoolKey="queryThreadPoolKey",fallbackMethod = "queryFallback",

        commandProperties = {

                @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100"),//指定多久超时，单位毫秒。超时进fallback

                @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "3"),//判断熔断的最少请求数，默认是10；只有在一个统计窗口内处理的请求数量达到这个阈值，才会进行熔断与否的判断

                @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),//判断熔断的阈值，默认值50，表示在一个统计窗口内有50%的请求处理失败，会触发熔断

        },

        threadPoolProperties = {

                @HystrixProperty(name = "coreSize", value = "30"),

                @HystrixProperty(name = "maxQueueSize", value = "100"),

                @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),

                @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),

                @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),

                @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "100000")

        })

public List<User> query(){

    return restTemplate.getForObject("http://service-user/user/query",List.class);

}

public List<User> queryFallback(){

    List<User> list = new ArrayList<>();

    User user = new User();

    user.setId("1000");

    user.setName("queryFallback");

    list.add(user);

    return list;

}

}


启动：service-ribbon 工程，当我们访问[http://127.0.0.1:8764/user/query](http://127.0.0.1:8764/user/query),浏览器显示： 

[

  {

    "id": "id0",

    "name": "testname0"

  },

  {

    "id": "id1",

    "name": "testname1"

  },

  {

    "id": "id2",

    "name": "testname2"

>  }
>]

此时关闭 service-user工程，当我们再访问[http://127.0.0.1:8764/user/query](http://127.0.0.1:8764/user/query),浏览器会显示： 

[

  {

    "id": "1000",

    "name": "queryFallback"

  }

>]

这就说明当 service-user 工程不可用的时候，service-ribbon调用 service-user的API接口时，会执行快速失败，直接返回一组字符串，而不是等待响应超时，这很好的控制了容器的线程阻塞。 

## **2：在Feign中使用断路器**

Feign是自带断路器的，在D版本的Spring Cloud中，它没有默认打开。需要在配置文件中配置打开它，在配置文件加以下代码： 

feign:

  hystrix:

>    enabled: true

基于service-feign工程进行改造，只需要在FeignClient的UserService接口的注解中加上fallback的指定类就行了： 

@FeignClient(value="service-user",fallback = UserServiceFallback.class)

public interface UserService {

    @RequestMapping(value="/user/query",method = RequestMethod.*GET*)

   public List<User> query();

}

* 123456

UserServiceFallback需要实现UserService 接口，并注入到Ioc容器中，代码如下： 

@Component

public class UserServiceFallback implements UserService {

@Override

public List<User> query() {

    List<User> list = new ArrayList<>();

    User user = new User();

    user.setId("1000");

    user.setName("feignFallback");

    list.add(user);

    return list;

}

}

启动四servcie-feign工程，浏览器打开[http://127.0.0.1:8765/user/query](http://127.0.0.1:8765/user/query)注意此时service-user工程没有启动，网页显示： 

[

  {

    "id": "1000",

    "name": "feignFallback",

    "date": null

  }

]

这证明断路器起到作用了。 

五、Hystrix Dashboard (断路器：Hystrix 仪表盘) 

基于service-ribbon 改造，Feign的改造和这一样。 

首选在pom.xml引入spring-cloud-starter-hystrix-dashboard的起步依赖： 

<dependency>

            <groupId>org.springframework.boot</groupId>

            <artifactId>spring-boot-starter-actuator</artifactId>

        </dependency>

        <dependency>

            <groupId>org.springframework.cloud</groupId>

            <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>

        </dependency>

在主程序启动类中加入@EnableHystrixDashboard注解，开启hystrixDashboard： 

@SpringBootApplication

@EnableDiscoveryClient

@EnableHystrix

@EnableHystrixDashboard

public class ServiceRibbonApplication {

    public static void main(String[] args) {

        SpringApplication.run(ServiceRibbonApplication.class, args);

    }

    @Bean

    @LoadBalanced

    RestTemplate restTemplate() {

        return new RestTemplate();

    }

}

## 3：Hystrix Dashboard (断路器：Hystrix 仪表盘)

打开浏览器：访问[http://localhost:8764/hystrix](http://localhost:8764/hystrix),界面如下： 

![图片](https://uploader.shimo.im/f/xqvftoZU5Q09yDlW.png!thumbnail)

点击monitor stream，进入下一个界面，访问：[http://127.0.0.1:8764/user/query](http://127.0.0.1:8764/user/query)

此时会出现监控界面： 

![图片](https://uploader.shimo.im/f/poYbSvXKsakNx1KN.png!thumbnail)



# 三：[Hystrix](https://github.com/Netflix/Hystrix)流程图 

下面的流程图展示了当使用Hystrix的依赖请求，Hystrix是如何工作的。

![图片](https://uploader.shimo.im/f/z676OD8YBNgwwywj.png!thumbnail)

 

​ 下面将更详细的解析每一个步骤都发生哪些动作： 

## **1.构建一个****HystrixCommand****或者****HystrixObservableCommand****对象。**

**第一步就是构建一个****HystrixCommand****或者****HystrixObservableCommand****对象，该对象将代表你的一个依赖请求，向构造函数中传入请求依赖所需要的参数**。 

如果构建HystrixCommand中的依赖返回单个响应，例如： 

HystrixCommand command = **new** HystrixCommand(arg1, arg2);

如果依赖需要返回一个Observable来发射响应，就需要通过构建HystrixObservableCommand对象来完 成，例如： 

* HystrixObservableCommand command = **new** HystrixObservableCommand(arg1, arg2);
## **2.执行命令**

* 有4种方式可以执行一个Hystrix命令。 
  * execute()—该方法是阻塞的，从依赖请求中接收到单个响应（或者出错时抛出异常）。
  * queue()—从依赖请求中返回一个包含单个响应的Future对象。
  * observe()—订阅一个从依赖请求中返回的代表响应的Observable对象。
  * toObservable()—返回一个Observable对象，只有当你订阅它时，它才会执行Hystrix命令并发射响应。

K             value   = command.execute();

Future<K>     fValue  = command.queue();

Observable<K> ohValue = command.observe();         *//hot observable*

* Observable<K> ocValue = command.toObservable();    *//cold observable*

**同步调用方法****execute()****实际上就是调用****queue().get()****方法，****queue()****方法的调用的是****toObservable().toBlocking().toFuture()****.也就是说，最终每一个HystrixCommand都是通过Observable来实现的，即使这些命令仅仅是返回一个简单的单个值。** 

## **3.响应是否被缓存**

* 如果这个命令的请求缓存已经开启，并且本次请求的响应已经存在于缓存中，那么就会立即返回一个包含缓存响应的Observable（下面将Request Cache部分将对请求的cache做讲解）。 
## **4.回路器是否打开**

当命令执行时，Hystrix会检查回路器是否被打开。 

**如果回路器被打开（或者tripped），那么Hystrix就不会再执行命名，而是直接路由到第****8****步**，获取fallback方法，并执行fallback逻辑。 

* 如果回路器关闭，那么将进入第5步，检查是否有足够的容量来执行任务。（其中容量包括线程池的容量，队列的容量等等）。 
## **5.线程池、队列、信号量是否已满**

* 如果与该命令相关的线程池或者队列已经满了，那么Hystrix就不会再执行命令，而是立即跳到第8步,执行fallback逻辑。 
## **6.HystrixObservableCommand.construct() 或者 HystrixCommand.run()**

* 在这里，Hystrix通过你写的方法逻辑来调用对依赖的请求，通过下列之一的调用： 
  * HystrixCommand.run()—返回单个响应或者抛出异常。

HystrixObservableCommand.construct() —返回一个发射响应的Observable或者发送一个onError()的通知。

如果run（）或construct（）方法超出命令的超时值，则线程将抛出TimeoutException（如果命令本身未在其自己的线程中运行，则将抛出单独的计时器线程）。

在这种情况下，Hystrix将响应路由到8.获取回退，如果该方法不取消/中断，它将丢弃最终返回值run（）或construct（）方法。

请注意，没有办法强制潜在的线程停止工作 - 最好的Hystrix可以在JVM上执行的操作是将其抛出InterruptedException。

如果由Hystrix包装的工作不遵守InterruptedExceptions，则Hystrix线程池中的线程将继续其工作，尽管客户端已经收到TimeoutException。

这种行为可以使Hystrix线程池饱和，尽管负载“正确脱落”。

大多数Java HTTP客户端库不解释InterruptedExceptions。

因此，请确保在HTTP客户端上正确配置连接和读/写超时。

如果该命令没有抛出任何异常并且它返回了响应，则Hystrix在执行一些日志记录和度量报告后返回此响应。

在run（）的情况下，Hystrix返回一个Observable，它发出单个响应，然后发出onCompleted通知;

在construct（）的情况下，Hystrix返回由construct（）返回的相同Observable。

## **7.计算回路指标[Circuit Health]**

Hystrix会报告成功、失败、拒绝和超时的指标给回路器，回路器包含了一系列的**滑动窗口**数据，并通过该数据进行统计。 

* 它使用这些统计数据来决定回路器是否应该熔断，如果需要熔断，将在一定的时间内不在请求依赖[短路请求]（译者：这一定的时候可以通过配置指定），当再一次检查请求的健康的话会重新关闭回路器。 
## **8.获取FallBack**

* 当命令执行失败时，Hystrix会尝试执行自定义的Fallback逻辑： 
  * 当construct()或者run()方法执行过程中抛出异常。
  * 当回路器打开，命令的执行进入了熔断状态。
  * 当命令执行的线程池和队列或者信号量已经满容。
  * 命令执行超时。

**写一个fallback方法，提供一个不需要网络依赖的通用响应，从内存缓存或者其他的静态逻辑获取数据。如果再fallback内必须需要网络的调用，更好的做法是使用另一个****HystrixCommand****或者****HystrixObservableCommand****。 **

如果你的命令是继承自HystrixCommand，那么可以通过实现HystrixCommand.getFallback()方法返回一个单个的fallback值。 

如果你的命令是继承自HystrixObservableCommand，那么可以通过实现HystrixObservableCommand.resumeWithFallback()方法返回一个Observable，并且该Observable能够发射出一个fallback值。 

Hystrix会把fallback方法返回的响应返回给调用者。 

如果你没有为你的命令实现fallback方法，那么当命令抛出异常时，Hystrix仍然会返回一个Observable，但是该Observable并不会发射任何的数据，并且会立即终止并调用onError()通知。通过这个onError通知，可以将造成该命令抛出异常的原因返回给调用者。 

失败或不存在回退的结果将根据您如何调用Hystrix命令而有所不同：

* execute()：抛出一个异常。
* queue()：成功返回一个Future，但是如果调用get()方法，将会抛出一个异常。
* observe()：返回一个Observable，当你订阅它时，它将立即终止，并调用onError()方法。
* toObservable()：返回一个Observable，当你订阅它时，它将立即终止，并调用onError()方法。
## **9.返回成功的响应**

* 如果Hystrix命令执行成功，它将以Observable形式返回响应给调用者。根据你在第2步的调用方式不同，在返回Observablez之前可能会做一些转换。 

![图片](https://uploader.shimo.im/f/2gaxJPOpXHMA7QpX.png!thumbnail)

* execute()：通过调用queue()来得到一个Future对象，然后调用get()方法来获取Future中包含的值。
* queue()：将Observable转换成BlockingObservable，在将BlockingObservable转换成一个Future。
* observe()：订阅返回的Observable，并且立即开始执行命令的逻辑，
* toObservable()：返回一个没有改变的Observable，你必须订阅它，它才能够开始执行命令的逻辑。
# 四：断路器

下图显示了HystrixCommand或HystrixObservableCommand如何与HystrixCircuitBreaker及其逻辑和决策流程进行交互，包括计数器在断路器中的行为方式。

![图片](https://uploader.shimo.im/f/pmVBSW6tfgEmlRJ6.png!thumbnail)

 

回路器打开和关闭有如下几种情况： 

* 假设回路中的请求满足了一定的阈值（HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()）
* 假设错误发生的百分比超过了设定的错误发生的阈值HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()
* 回路器状态由CLOSE变换成OPEN
* 如果回路器打开，所有的请求都会被回路器所熔断。
* 一定时间之后HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()，下一个的请求会被通过（处于半打开状态），如果该请求执行失败，回路器会在睡眠窗口期间返回OPEN，如果请求成功，回路器会被置为关闭状态，重新开启1步骤的逻辑。

Hystrix的熔断器实现在HystrixCircuitBreaker类中，比较重要的几个参数如下： 

1、circuitBreaker.enabled 

熔断器是否启用，默认是true 

2、circuitBreaker.forceOpen 

熔断器强制打开，始终保持打开状态，默认是false 

3、circuitBreaker.forceClosed 

熔断器强制关闭，始终保持关闭状态，默认是false 

4、circuitBreaker.requestVolumeThreshold 

滑动窗口内（10s）的请求数阈值，只有达到了这个阈值，才有可能熔断。默认是20，如果这个时间段只有19个请求，就算全部失败了，也不会自动熔断。 

5、circuitBreaker.errorThresholdPercentage 

错误率阈值，默认50%，比如（10s）内有100个请求，其中有60个发生异常，那么这段时间的错误率是60，已经超过了错误率阈值，熔断器会自动打开。 

6、circuitBreaker.sleepWindowInMilliseconds 

熔断器打开之后，为了能够自动恢复，每隔默认5000ms放一个请求过去，试探所依赖的服务是否恢复。 

* 在最新代码中，已经弃用了allowRequest()，取而代之的是attemptExecution()方法。

![图片](https://uploader.shimo.im/f/5GSekr3K3MoTdurv.png!thumbnail)

和allowRequest()方法相比，唯一改进的地方是通过compareAndSet修改状态值。通过attemptExecution()方法的返回值决定执行正常逻辑，还是降级逻辑。

1、如果circuitBreaker.forceOpen=true，说明熔断器已经强制开启，所有请求都会被熔断。

2、如果circuitBreaker.forceClosed =true，说明熔断器已经强制关闭，所有请求都会被放行。

3、circuitOpened默认-1，用以保存最近一次发生熔断的时间戳。

4、如果circuitOpened不等于-1，说明已经发生熔断，通过isAfterSleepWindow()判断当前是否需要进行试探。

![图片](https://uploader.shimo.im/f/3ATZwuZ30f8u05eu.png!thumbnail)

这里就是熔断器自动恢复的逻辑，如果当前时间已经超过上次熔断的时间戳 + 试探窗口5000ms，则进入if分支，通过compareAndSet修改变量status，竞争试探的能力。其中status代表当前熔断器的状态，包含CLOSED, OPEN, HALF_OPEN，只有试探窗口之后的第一个请求可以执行正常逻辑，且修改当前状态为HALF_OPEN，进入半熔断状态，其它请求执行compareAndSet(Status.OPEN, Status.HALF_OPEN)时都返回false，执行降级逻辑。

5、如果试探请求发生异常，则执行markNonSuccess() 

![图片](https://uploader.shimo.im/f/in5BECAULjoKMCLj.png!thumbnail)

通过compareAndSet修改status为熔断开启状态，并更新当前熔断开启的时间戳。 

6、如果试探请求返回成功，则执行markSuccess() 

![图片](https://uploader.shimo.im/f/eRyZD26j12Q7i3hJ.png!thumbnail)

通过compareAndSet修改status为熔断关闭状态，并重置接口统计数据和circuitOpened标识为-1，后续请求开始执行正常逻辑。 

**说了这么多，如何实现自动熔断还没提到，在Hystrix内部有一个Metric模块，专门统计每个Command的执行状态，包括成功、失败、超时、线程池拒绝等，在熔断器的中subscribeToStream()方法中，通过订阅数据流变化，实现函数回调，当有新的请求时，数据流发生变化，触发回调函数onNext **

![图片](https://uploader.shimo.im/f/cZhuRT3ORQkSOv1g.png!thumbnail)

在onNext方法中，参数hc保存了当前接口在前10s之内的请求状态（请求总数、失败数和失败率）,其主要逻辑是判断请求总数是否达到阈值requestVolumeThreshold，失败率是否达到阈值errorThresholdPercentage，如果都满足，说明接口的已经足够的不稳定，需要进行熔断，则设置status为熔断开启状态，并更新circuitOpened为当前时间戳，记录上次熔断开启的时间。 


# 五：隔离 

Hystrix采用**舱壁模式**来隔离相互之间的依赖关系，并限制对其中任何一个的并发访问。

![图片](https://uploader.shimo.im/f/am3OoENa7aUlbHVt.png!thumbnail)

 

## **线程和线程池**

客户端（第三方包、网络调用等）会在单独的线程执行，会与调用的该任务的线程进行隔离，以此来防止调用者调用依赖所消耗的时间过长而阻塞调用者的线程。 

* [Hystrix uses separate, per-dependency thread pools as a way of constraining any given dependency so latency on the underlying executions will saturate the available threads only in that pool] 

![图片](https://uploader.shimo.im/f/T0HUnivWEC8IVBIr.png!thumbnail)

您可以在不使用线程池的情况下防止出现故障，但是这要求客户端必须能够做到快速失败（网络连接/读取超时和重试配置），并始终保持良好的执行状态。 

Netflix，设计Hystrix，并且选择使用线程和线程池来实现隔离机制，有以下几个原因： 

* 很多应用会调用多个不同的后端服务作为依赖。
* 每个服务会提供自己的客户端库包。
* 每个客户端的库包都会不断的处于变更状态。
* [Client library logic can change to add new network calls]
* 每个客户端库包都可能包含重试、数据解析、缓存等等其他逻辑。
* 对用户来说，客户端库往往是“黑盒”的，对于实现细节、网络访问模式。默认配置等都是不透明的。
* [In several real-world production outages the determination was “oh, something changed and properties should be adjusted” or “the client library changed its behavior.]
* 即使客户端本身没有改变，服务本身也可能发生变化，这些因素都会影响到服务的性能，从而导致客户端配置失效。
* 传递依赖可以引入其他客户端库，这些客户端库不是预期的，也许没有正确配置。
* 大部分的网络访问是同步执行的。
* 客户端代码中也可能出现失败和延迟，而不仅仅是在网络调用中。

![图片](https://uploader.shimo.im/f/xBKGp9fNV9YnToXV.png!thumbnail)

**使用线程池的好处**

* 通过线程在自己的线程池中隔离的好处是： 
  * 该应用程序完全可以不受失控的客户端库的威胁。即使某一个依赖的线程池已满也不会影响其他依赖的调用。
  * 应用程序可以低风险的接受新的客户端库的数据，如果发生问题，它会与出问题的客户端库所隔离，不会影响其他依赖的任何内容。
  * 当失败的客户端服务恢复时，线程池将会被清除，应用程序也会恢复，而不至于使得我们整个Tomcat容器出现故障。
  * 如果一个客户端库的配置错误，线程池可以很快的感知这一错误（通过增加错误比例，延迟，超时，拒绝等），并可以在不影响应用程序的功能情况下来处理这些问题（可以通过动态配置来进行实时的改变）。
  * 如果一个客户端服务的性能变差，可以通过改变线程池的指标（错误、延迟、超时、拒绝）来进行属性的调整，并且这些调整可以不影响其他的客户端请求。
  * 除了隔离的优势之外，拥有专用的线程池可以提供内置的请求任务的并发性，可以在同步客户端上构建异步门面。

简而言之，**由线程池提供的隔离功能可以使客户端库和子系统性能特性的不断变化和动态组合得到优雅的处理，而不会造成中断。 **

*注意*：虽然单独的线程提供了隔离，但您的底层客户端代码也应该有超时和/或响应线程中断，而不能让Hystrix的线程池处于无休止的等待状态。 

**线程池的缺点**

线程池最主要的缺点就是增加了CPU的计算开销，每个命令都会在单独的线程池上执行，这样的执行方式会涉及到命令的排队、调度和上下文切换。 

* Netflix在设计这个系统时，决定接受这个开销的代价，来换取它所提供的好处，并且认为这个开销是足够小的，不会有重大的成本或者是性能影响。 

**线程成本**

Hystrix在子线程执行construct()方法和run()方法时会计算延迟，以及计算父线程从端到端的执行总时间。所以，你可以看到Hystrix开销成本包括（线程、度量，日志，断路器等）。 

Netflix API每天使用线程隔离的方式处理10亿多的Hystrix Command任务，每个API实例都有40多个线程池，每个线程池都有5-20个线程（大多数设置为10） 

* 下图显示了一个HystrixCommand在单个API实例上每秒执行60个请求（每个服务器每秒执行大约350个线程执行总数）： 

![图片](https://uploader.shimo.im/f/9ZxHT3jiMW4kaS3O.png!thumbnail)

在中间位置（或者下线位置）不需要单独的线程池。 

在第90线上，单独线程的成本为3ms。 

在第99线上，单独的线程花费9ms。但是请注意，线程成本的开销增加远小于单独线程（网络请求）从2跳到28而执行时间从0跳到9的增加。 

对于大多数Netflix用例来说，这样的请求在90％以上的开销被认为是可以接受的，这是为了实现韧性的好处。 

**对于非常低延迟请求（例如那些主要触发内存缓存的请求），开销可能太高，在这种情况下，可以使用另一种方法，如信号量，虽然它们不允许超时，提供绝大部分的有点，而不会产生开销。然而，一般来说，开销是比较小的，以至于Netflix通常更偏向于通过单独的线程来作为隔离实现。 **

## 线程隔离-信号量

上面提到了线程池隔离的缺点，当依赖延迟极低的服务时，线程池隔离技术引入的开销超过了它所带来的好处。这时候可以使用信号量隔离技术来代替，通过设置信号量来限制对任何给定依赖的并发调用量。下图说明了线程池隔离和信号量隔离的主要区别：

![图片](https://uploader.shimo.im/f/N8TVqls8ceAcbcB8.png!thumbnail)

                            图片来源Hystrix官网[https://github.com/Netflix/Hystrix/wiki](https://github.com/Netflix/Hystrix/wiki)

使用线程池时，发送请求的线程和执行依赖服务的线程不是同一个，而使用信号量时，发送请求的线程和执行依赖服务的线程是同一个，都是发起请求的线程。

您可以使用信号量（或计数器）来限制对任何给定依赖项的并发调用数，而不是使用线程池/队列大小。这允许Hystrix在不使用线程池的情况下卸载负载，但它不允许超时和离开。如果您信任客户端而您只想减载，则可以使用此方法。HystrixCommand和HystrixObservableCommand支持2个地方的信号量：回退：当Hystrix检索回退时，它总是在调用Tomcat线程上执行此操作。执行：如果将属性execution.isolation.strategy设置为SEMAPHORE，则Hystrix将使用信号量而不是线程来限制调用该命令的并发父线程数。您可以通过定义可以执行多少并发线程的动态属性来配置信号量的这两种用法。您应该使用在调整线程池大小时使用的类似计算来调整它们的大小（以毫秒为单位返回的内存中调用可以在5000rps下执行，信号量仅为1或2 ......但默认值为10）。注意：如果依赖项与信号量隔离然后变为潜在的，则父线程将保持阻塞状态，直到基础网络调用超时。信号量拒绝将在限制被触发后开始，但填充信号量的线程无法离开。

由于Hystrix默认使用线程池做线程隔离，使用信号量隔离需要显示地将属性execution.isolation.strategy设置为ExecutionIsolationStrategy.SEMAPHORE，同时配置信号量个数，默认为10。客户端需向依赖服务发起请求时，首先要获取一个信号量才能真正发起调用，由于信号量的数量有限，当并发请求量超过信号量个数时，后续的请求都会直接拒绝，进入fallback流程。

**信号量隔离主要是通过控制并发请求量，防止请求线程大面积阻塞，从而达到限流和防止雪崩的目的。**

## 隔离总结

线程池和信号量都可以做线程隔离，但各有各的优缺点和支持的场景，对比如下：

| ** **   | **线程切换**   | **支持异步**   | **支持超时**   | **支持熔断**   | **限流**   | **开销**   | 
|:----|:----|:----|:----|:----|:----|:----|
| **信号量**   | **否**   | **否**   | **否**   | **是**   | **是**   | **小**   | 
| **线程池**   | **是**   | **是**   | **是**   | **是**   | **是**   | **大**   | 

线程池和信号量都支持熔断和限流。相比线程池，信号量不需要线程切换，因此避免了不必要的开销。但是信号量不支持异步，也不支持超时，也就是说当所请求的服务不可用时，信号量会控制超过限制的请求立即返回，但是已经持有信号量的线程只能等待服务响应或从超时中返回，即可能出现长时间等待。线程池模式下，当超过指定时间未响应的服务，Hystrix会通过响应中断的方式通知线程立即结束并返回。

## 请求合并 

您可以使用请求合并器（HystrixCollapser是抽象父代）来提前发送HystrixCommand，通过该合并器您可以将多个请求合并为一个后端依赖项调用。 

下面的图展示了两种情况下的线程数和网络连接数，第一张图是不使用请求合并，第二张图是使用请求合并（假定所有连接在短时间窗口内是“并发的”，在这种情况下是10ms）。 

![图片](https://uploader.shimo.im/f/VQvULGQXOgAhoVkq.png!thumbnail)

**为什么使用请求合并**

* 事情请求合并来减少执行并发HystrixCommand请求所需要的线程数和网络连接数。请求合并以自动方式执行的，不需要代码层面上进行批处理请求的编码。 

全局上下文（所有的tomcat线程）

理想的合并方式是在全局应用程序级别来完成的，以便来自任何用户的任何Tomcat线程的请求都可以一起合并。 

例如，如果将HystrixCommand配置为支持任何用户请求获取影片评级的依赖项的批处理，那么当同一个JVM中的任何用户线程发出这样的请求时，Hystrix会将该请求与其他请求一起合并添加到同一个JVM中的网络调用。 

  * 请注意，合并器会将一个HystrixRequestContext对象传递给合并的网络调用，为了使其成为一个有效选项，下游系统必须处理这种情况。 

用户请求上下文（单个tomcat线程）

如果将HystrixCommand配置为仅处理单个用户的批处理请求，则Hystrix仅仅会合并单个Tomcat线程的请求。 

  * 例如，如果一个用户想要加载300个影片的标签，Hystrix能够把这300次网络调用合并成一次调用。 

对象建模和代码的复杂性

有时候，当你创建一个对象模型对消费的对象而言是具有逻辑意义的，这与对象的生产者的有效资源利用率不匹配。 

例如，给你300个视频对象，遍历他们，并且调用他们的getSomeAttribute()方法，但是如果简单的调用，可能会导致300次网络调用（可能很快会占满资源）。 

有一些手动的方法可以解决这个问题，比如在用户调用getSomeAttribute()方法之前，要求用户声明他们想要获取哪些视频对象的属性，以便他们都可以被预取。 

或者，您可以分割对象模型，以便用户必须从一个位置获取视频列表，然后从其他位置请求该视频列表的属性。 

这些方法可以会使你的API和对象模型显得笨拙，并且这种方式也不符合心理模式与使用模式（译者：不太懂什么意思）。由于多个开发人员在代码库上工作，可能会导致低级的错误和低效率开发的问题。因为对一个用例的优化可以通过执行另一个用例和通过代码的新路径来打破。 

通过将合并逻辑移到Hystrix层，不管你如何创建对象模型，调用顺序是怎样的，或者不同的开发人员是否知道是否完成了优化或者是否完成。 

  * getSomeAttribute（）方法可以放在最适合的地方，并以任何适合使用模式的方式被调用，并且合并器会自动将批量调用放置到时间窗口。 
## 请求Cache 

HystrixCommand和HystrixObservableCommand实现可以定义一个缓存键，然后用这个缓存键以并发感知的方式在请求上下文中取消调用（不需要调用依赖即可以得到结果，因为同样的请求结果已经按照缓存键缓存起来了）。 

以下是一个涉及HTTP请求生命周期的示例流程，以及在该请求中执行工作的两个线程：

![图片](https://uploader.shimo.im/f/El7anE913KEaJEo0.png!thumbnail)

 

请求cache的好处有： 

* 不同的代码路径可以执行Hystrix命令，而不用担心重复的工作。

这在许多开发人员实现不同功能的大型代码库中尤其有用。 

例如，多个请求路径都需要获取用户的Account对象，可以像这样请求： 

Account account = **new** UserGetAccount(accountId).execute();

*//or*

Observable<Account> accountObservable = **new** UserGetAccount(accountId).observe();

Hystrix RequestCache将只执行一次底层的run（）方法，执行HystrixCommand的两个线程都会收到相同的数据，尽管实例化了多个不同的实例。 

* 整个请求的数据检索是一致的。

每次执行该命令时，不再会返回一个不同的值（或回退），而是将第一个响应缓存起来，后续相同的请求将会返回缓存的响应。 

* 消除重复的线程执行。

由于请求缓存位于construct（）或run（）方法调用之前，Hystrix可以在调用线程执行之前取消调用。 

如果Hystrix没有实现请求缓存功能，那么每个命令都需要在构造或者运行方法中实现，这将在一个线程排队并执行之后进行。 

# 附：Hystrix源码图

![图片](https://uploader.shimo.im/f/xhxSiQnwvKYwN2nL.png!thumbnail)


