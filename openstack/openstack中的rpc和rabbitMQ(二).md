# 理解openstack当中的rpc远程调用(二)

## 你需要知道的AMQP

RPC只是定义了一个通信接口，其底层的实现可以各不相同，可以是 socket，也可以是今天要讲的 AMQP。  
AMQP(Advanced Message Queuing Protocol)是一种基于队列的可靠消息服务协议，作为一种通信协议，AMQP同样存在多个实现，
如Apache Qpid，RabbitMQ等。

关于AMQP的概念：
- Publisher：消息发布者
- Receiver：消息接收者，在RabbitMQ中叫订阅者：Subscriber。
- Queue：用来保存消息的存储空间，消息没有被receiver前，保存在队列中。
- Exchange：用来接收Publisher发出的消息，根据Routing key 转发消息到对应的Message Queue中，
  至于转到哪个队列里，这个路由算法又由exchange type决定的。
  > 1). direct：消息路由到满足此条件的队列中(queue,可以有多个)： routing key = binding key
  > 
  > 2). topic：消息路由到满足此条件的队列中(queue,可以有多个)：routing key 匹配 binding pattern. binding pattern是类似
  > 正则表达式的字符串，可以满足复杂的路由条件。
  > 
  > 3). fanout：消息路由到多有绑定到该exchange的队列中。
  > 
- binding ：binding是用来描述exchange和queue之间的关系的概念，一个exchang可以绑定多个队列，
  这些关系由binding建立。前面说的binding key /binding pattern也是在binding中给出。
  
这些概念的关系如下图所示：

![img_14.png](img_14.png)

关于AMQP，有几下几点值得注意：
- 每个receiver/subscriber 在接收消息前都需要创建binding；
- 一个队列可以有多个receiver，队列里的一个消息只能发给一个receiver；
- 一个消息可以被发送到一个队列中，也可以被发送到多个多列中。多队列情况下，
一个消息可以被多个receiver收到并处理。Openstack RPC中这两种情况都会用到。
  
## oslo_messaging中的rpc
OpenStack RPC 模块提供了 rpc.call，rpc.cast, rpc.fanout_cast 三种 RPC 调用方法，发送和接收 RPC 请求。
- rpc.call 发送 RPC 同步请求并返回请求处理结果；
- rpc.cast 发送 RPC 异步请求，与 rpc.call 不同之处在于，不需要请求处理结果的返回；
- rpc.fanout_cast 用于发送 RPC 广播信息无返回结果。

rpc.call 和 rpc.cast 从实现代码上看，他们的区别很小，就是call调用时候会带有`wait_for_reply=True`参数，而cast不带。

oslo_messaging中rpc的元素：
#### transport
RPC功能的底层实现方法，描述了rabbitmq的消息队列的访问路径。   
当使用的是 Rabbitmq，那在 nova.conf中应该有一行transport_url的配置，配置了连接rabbitmq的user，passwd，主机，端口。

```python
transport_url=rabbit://user:passwd@host:5672
```

#### target
指定RPC topic交换机的匹配信息和绑定主机。   
target用来表述 RPC 服务器监听topic，server名称和server监听的exchange，是否广播fanout。
```python
class Target(object):
      def __init__(self, exchange=None, topic=None, namespace=None,
               version=None, server=None, fanout=None,
               legacy_namespaces=None):
      self.exchange = exchange
      self.topic = topic
      self.namespace = namespace
      self.version = version
      self.server = server
      self.fanout = fanout
      self.accepted_namespaces = [namespace] + (legacy_namespaces or [])
```
rpc server 要获取消息，需要定义target，就像一个门牌号一样。
```python
class ConductorManager(manager.Manager):
    ...
    target = messaging.Target(version='3.0')
```
```python
class ComputeTaskManager(base.Base):
    ...
    target = messaging.Target(namespace='compute_task', version='1.23')
```

rpc client 要发送消息，也需要有target，说明消息要发到哪去。
```python
class ComputeTaskAPI(object):
    ...
    def __init__(self):
      super(ComputeTaskAPI, self).__init__()
      target = messaging.Target(topic=RPC_TOPIC,            # 描述了发给哪个server
                                namespace='compute_task',   # 调用的具体的方法
                                version='1.0')
```

#### endpoints
是可供别人远程调用的对象
```python
# nova.service
class Service(service.Service):
    ...
    def start(self):
        ...
        endpoints = [
            self.manager,
            baserpc.BaseRPCAPI(self.manager.service_name, self.backdoor_port)
        ]
        endpoints.extend(self.manager.additional_endpoints)
        ...
```
对于conductor server，这里endpoints就是 `nova/conductor/manager.py:ConductorManager()`

#### dispatcher
rpc server中的分发器。client端直接指定调用的方法名称，server端的dispatcher则能够在endpoints中找到对应的方法，
执行该方法，并将结果返回。
```python
# oslo_messaging/rpc/server.py
def get_rpc_server(transport, target, endpoints,
                   executor=None, serializer=None, access_policy=None):
    dispatcher = rpc_dispatcher.RPCDispatcher(endpoints, serializer,
                                              access_policy)
    return RPCServer(transport, target, dispatcher, executor)
```
dispatch通过反射，在endpoints中找到调用的方法：
```python
# oslo_messaging/rpc/dispatcher.py
class RPCDispatcher(dispatcher.DispatcherBase):
    ...
    def _do_dispatch(self, endpoint, method, ctxt, args):
        ctxt = self.serializer.deserialize_context(ctxt)
        new_args = dict()
        for argname, arg in args.items():
            new_args[argname] = self.serializer.deserialize_entity(ctxt, arg)
        func = getattr(endpoint, method)
        result = func(ctxt, **new_args)
        return self.serializer.serialize_entity(ctxt, result)
    ...
```
#### Serializer
在 python 对象和message(notification) 之间数据做序列化或是反序列化的基类，主要方法有四个：
- deserialize_context(ctxt) ：对字典变成 request contenxt. 
- deserialize_entity(ctxt, entity) ：对entity做反序列化，其中ctxt是已经deserialize过的，entity是要处理的。
- serialize_context(ctxt) ：将Request context变成字典类型
- serialize_entity(ctxt, entity) ：对entity做序列化，其中ctxt是已经deserialize过的，entity是要处理的。

#### executor
服务的运行方式，单线程或者多线程。每个notification listener都和一个executor绑定，来控制收到的notification如何分配。

#### rpc server 和rpc client 的四个重要方法
- `reset()`：Reset service.  
- `start()`：该方法调用后，server开始poll,从transport中接收message,然后转发给dispatcher.
  该message处理过程一直进行，直到stop方法被调用。executor决定server的IO处理策略。
  可能会是用一个新进程、新协程来做poll操作，或是直接简单的在一个循环中注册一个回调。
  同样，executor也决定分配message的方式，是在一个新线程中dispatch；
- `stop()`:当调用stop之后，新的message不会被处理。但是，server可能还在处理一些之前没有处理完的message,
  并且底层driver资源也还一直没有释放。
- `wait()`：在stop调用之后，可能还有message正在被处理，使用wait方法来阻塞当前进程，
  直到所有的message都处理完成。之后，底层的driver资源会释放。

## 模仿openstack中的rpc

<details> 
<summary><font size="4" color="orange">rpc server</font></summary>

```python
#coding:utf-8

from oslo_config import cfg
import oslo_messaging
import time 

class ServerControlEndpoint(object):
    
    def __init__(self, server):
        
        self.server = server
    
    def stop(self, ctx):
        if self.server:
            self.server.stop()

class AddEndpoint(object):
    
    def add(self, ctx, a, b):
        print("successfully reviced your message, bytedancer!")
        return a + b
        
        
# 从配置文件中加载transport_url。在openstack中，账号、密码、端口号等都是从配置文件中读取，支持可配置的
# 配置文件的内容通过oslo_config库读取.

opts = [cfg.StrOpt('url', default='0.0.0.0')]
CONF = cfg.CONF
CONF.register_opts(opts) 
CONF(default_config_files=['my.conf'])

# transport_url是指定实现RPC的底层技术，可以使rabbitmq，也可以是别的技术
# 从my.conf文件中读取到该URL。
transport_url = CONF.url
transport = oslo_messaging.get_rpc_transport(cfg.CONF, transport_url)
print("transport: ", transport_url)

# target用来指定该rpc server监听在哪些队列上。
# target指定了2个参数：topic和server。
target = oslo_messaging.Target(topic='test', server='server1')

# 可供别人调用的方法类
endpoints = [ServerControlEndpoint(None), AddEndpoint()]

# 创建Server对象时，需要指定Transport、Target和一组endpoint
server = oslo_messaging.get_rpc_server(transport, target, endpoints, executor='threading')

try:
    server.start()
    print("The Server has started!")
    while True:
        time.sleep(0.5)
except KeyboardInterrupt:
    print("Stopping server...")
    server.stop()
```

</details>

<details> 
<summary><font size="4" color="blue">rpc client</font></summary>

```python
#coding:utf-8
# 
from oslo_config import cfg
import oslo_messaging

opts = [cfg.StrOpt('url', default='helloworld')]
CONF = cfg.CONF
CONF.register_opts(opts)
CONF(default_config_files=['my.conf'])

transport_url = CONF.url
transport = oslo_messaging.get_rpc_transport(cfg.CONF,transport_url)
print("transport: ", transport_url)

target = oslo_messaging.Target(topic='test')
client = oslo_messaging.RPCClient(transport, target)
r = client.call({}, 'add', a = 2, b = 3)
print("result :",r)

#Target对象的属性在RPCClient对象构造之后，还可以通过prepare()方法修改，
# #可以修改的属性包括exchange,topic,namespace,version，server,fanout和retry。
# #修改后的target属性只在这个prepare()方法返回的对象中有效。

cctxt = client.prepare(version='1.0')
r_two = cctxt.call({},'add',a =2, b = 3)
print("result_two: ", r_two)
```
</details>

配置文件 `my.conf`，提供了rpc底层使用的rabbitMQ的请求路径。
```shell
[DEFAULT]

url = 'rabbit://bytedance:iaas2022@127.0.0.1:5672/'
```

其中，`bytendance`和`iaas2022`是创建的消息队列用户名和密码。
```shell
# 提前创建rabbit的用户名和密码，并赋予高权限
rabbitmqctl add_user bytendance iaas2022
rabbitmqctl set_permissions bytendance ".*" ".*" ".*"
```

调用结果：
![img_15.png](img_15.png)

## RabbitMQ监控
> 启用 RabbitMQ 管理 plugin

默认安装中，我们只能用命令rabbitmqctl 监控 RabbitMQ，比如：    
rabbitmqctl list_queues   
rabbitmqctl list_exchanges    
等子命令。这种方式不太直观，效率不高。

好在 RabbitMQ 有一个管理 plugin，提供了图形管理界面，可以在运行 RabbitMQ 的节点（一般是控制节点）执行下面的命令启用。
```shell
rabbitmq-plugins enable rabbitmq_management
```

然后还需要创建一个 用户，用来登录管理控制台了。
```shell
rabbitmqctl add_user  user_admin  passwd_admin

rabbitmqctl set_user_tags user_admin administrator

rabbitmqctl set_permissions -p / user_admin ".*" ".*" ".*"
```

然后就可以用 user_admin（密码 passwd_admin）登录了，地址是 

`http://server-name:15672/`

![img_16.png](img_16.png)

> 最简单高效的监控方法

Web 控制台会展示很多 RabbitMQ 信息，但最最重要的就一个：**Unacked Message**。
这个数据会直接显示在登录之后的 Overview 标签中，第一眼就能看到。

![img_17.png](img_17.png)

Unacked Message 指的是还没有被处理的消息。正常情况下，这个值应该为 0。如不是 0，
并且持续增长，那你就得注意了，这意味着 RabbitMQ 出现了问题，队列开始积压，消息开始堆积，是一个严重的信号。

这个时候就可以点开 Overview 后面的标签，查看到底消息是在哪个或者哪些 Connection，Channel，
Exchange，Queues 中堆积，进而分析问题的根源并解决。

![img_18.png](img_18.png)

![img_19.png](img_19.png)

> 一个真实案例

1. 客户的 OpenStack 在正常运行了一个月后突然挂了。

2. 日志分析发现 nova，neutron 等模块都报告找不到相关的 queue。因为多个模块的日志都指向 RabbitMQ，看来 RabbitMQ 有最大嫌疑。

3. RabbitMQ 日志中 Error 已经在持续刷屏，但信息很笼统。这时 RabbitMQ 已经处于无法工作的状态，只能重启 RabbitMQ。

4. RabbitMQ 重启后，OpenStack 自动恢复。

5. 打开 RabbitMQ Web 控制台，发现 Unacked Message > 0。

6. 观察一段时间发现 Unacked Message 以固定的速度持续增长。

7. 定位 Message 增长的原因，发现均来自 Ceilometer 相关的 Queue。

8. 检查 Ceilometer，发现了一个配置错误，导致 Ceilometer 发送到 Queue 的某些数据不会被处理。

9. 修改配置，重启 Ceilometer，Unacked Message 开始下降，最后保持为 0。 
  
    这个问题就像内存泄漏一样，Unacked Message 逐渐积累，最终压跨了整个 OpenStack。

## Notification (未完待续)


####参考：   
- [利用 OpenStack 全面理解 RPC 远程调用](https://iswbm.com/136.html)
- [基于Rabbit实现的RPC](https://www.cnblogs.com/goldsunshine/p/8665456.html)
- [模仿OpenStack写自己的RPC](https://www.cnblogs.com/goldsunshine/p/10205058.html)
- [Openstack基础组件之oslo.messaging](https://www.jianshu.com/p/4847e213b626)      
- [每天五分钟- 最简单的 RabbitMQ 监控方法](https://mp.weixin.qq.com/s?__biz=MzIwMTM5MjUwMg==&mid=2653587480&idx=1&sn=d02be878adcbca6bba898c367ac9e46e&chksm=8d308001ba470917b3b3c7564dba540d6d0799ebcdf4851323c2e71815cec1c78964b7110134&scene=21#wechat_redirect)
- [OpenStack RPC框架解析](https://www.cnblogs.com/luohaixian/p/11084279.html)
- [记一次OpenStack RPC断连问题分析过程](https://www.jianshu.com/p/c91ef649e588)