本文件夹下包含有如下知识：

- Java 基础知识
- 计算机网络
- Mysql
- 操作系统
- Java 并发
- JVM
- redis
- spring

**注意，由于文件中的图片使用的是相对路径而不是网络路径，可以下载整个项目后查阅。**

## 项目背景

项目实践内容是开发教师实验室预约系统，本系统是一款基于 Springboot + Vue 的预约系统，实现了教师能够基于课程条件预约一段时间实验室的功能。

系统分为管理员和教师两种角色。管理员能够对实验室信息、教师信息和预约信息进行管理，教师能够对个人信息、个人课程信息和个人预约信息进行管理。

数据库设计主要是对预约信息进行存储，预约信息包含有课程，实验室，预约的星期几，预约的第几节课，预约的哪几周。为了满足三范式，使用两个表进行存储，course_time, course_week。

course_time 表存储课程，实验室，预约的星期几，预约的第几节课。course_week 存储的是 course_time 的主键和对应预约的第几周。

具体的逻辑就是 course_time 为课程选择一个实验室，具体到一周的星期几的第几节课，而 course_week 就是预约的哪几周。

根据前端的要求，需要查询具体教师的预约信息或者教师课程的预约信息，对 course_time 的教师id 和课程id 建立联合索引；其次对实验室的预约信息查询，对 course_time 的实验室 id 建立普通索引；最后是某个范围的周次的预约信息，对 course_week 的周次建立普通索引，以达到加快查询效率的目的。

## 项目技术

- 使用 `Restful` 风格的接口，用一个 URI 表示一个资源，并通过 GET、POST、PUT、DELETE 操作资源。
- 前后端约定了创建 `ResultVO` 类用于封装数据，包含状态码，信息和数据，其中数据封装在 Map 中以键值对的形式返回。并且通过 `Jackson` 框架序列化所有属性。
- 由于 springboot 不会处理受检异常，所以显示捕获受检异常，转抛为非受检异常，并且自定义异常controller，创建方法捕获自定义异常，返回 JSON 数据。
- 使用 springboot 验证依赖检测前端请求参数，通过注解的方式进行简单的数值验证，如 `@NotNull`、`@Max/@Min`等，不符合的情况下抛出异常，并由自定义异常 controller 捕获对应的异常，返回 JSON 数据。
- 使用 spring 提供的安全框架，注入 `PasswordEncoder` 组件，对密码进行非对称加密入库，可以防止撞库或脱库时对用户密码的泄露。
- 使用 `token` 对身份进行验证，Restful 设计思想中，客户端不再保存用户状态，用户登录后直接将身份信息保存在 header 中，客户端和服务器的身份通过 `token` 中的信息来判别。同时 `token` 使用 Spring-Security 提供的非对称加密技术，可自定义密钥和盐值，通过 `@Value` 注解并结合 `SpEL`表达式，可以将`application.yml`配置中的信息注入。
- 使用三个 `Interceptor` 对用户身份进行拦截，首先通过 `LoginInterceptor` 的用户可以访问公共接口，通过 `adminInterceptor` 的用户可以访问管理员接口，通过 `teacherInterceptor` 的用户可以访问教师接口。拦截的方法就是通过判断用户 `header` 中的 `token` ，判断其中的身份，给予对应的权限。
- 使用 `Caffeine` 缓存技术对系统中的预约信息进行缓存，如对应教师的预约信息，对应实验室的预约信息。
- 使用 `Mybatis-Plus`持久层框架，可以通过实现 `BaseMapper` 接口，直接调用插入、查询等方法。
- 使用 MyBatis xml 的动态SQL查询实现数据按条件查询的功能。
- 在数据库设计阶段没有使用外键，而是使用的索引。级联操作在应用层用代码实现。

## 项目改进

- 使用caffine + redis二级缓存

  如果只使用 redis 来缓存，我们会有大量的请求到 redis，但是每次请求都是一样的，假如这部分数据就放在应用服务器本地，就省去了请求 redis 的网络开销，请求速度就会快很多。

  如果只使用 caffine 来做本地缓存，由于我们的应用服务的内存是有限的，单独为了缓存去扩展应用服务器是非常不划算的，所以，只使用本地缓存也有很大的局限性。

  **caffine 相比 redis 没有网络 IO 的消耗。**

  所以使用两个缓存，将热点数据放到本地缓存，作为一级缓存，将非热点数据放 redis 缓存，作为二级缓存。

- 采用分布式

  当前项目采用是将所有服务都部署到一个服务器上，这样在大量请求时很可能会出现宕机的情况，所以应该将服务进行剥离，将一个服务部署到一台服务器上，而多个服务之间采用 RPC 进行调用，RPC 的实现可以采用当前流行的 Dubbo 框架。 	

- Ngnix 负载均衡

  由于这个项目最重要的功能就是预约，所以可以将预约服务分布在多个服务器上，然后通过 Ngnix 做负载均衡，这样可以避免某个服务器的压力过大。

- 采用消息队列

  在某段时间可能预约的请求会激增，可以采用消息队列的形式，能够异步的对请求进行处理，即客户端发送请求后不立即返回提交成功，需要在消息队列中真正处理完之后再通知预约成功，从而减少响应所需的时间，可以采用 RocketMQ 或者 RabbitMQ

- 使用定时器

  一般来说实验室的预约会在一个学期的开学的时候，所以可以使用 springboot 的 `@scheduled` 来定时开发预约的操作。
