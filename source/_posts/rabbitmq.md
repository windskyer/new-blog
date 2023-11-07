---
title: Openstack之rabbitmq
date: 2016-04-21 10:22:00
tags: rabbitmq
categories: Openstack
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# rabbitmq 的工作流程和详细分析

## 1,rabbitmq 的 在openstack 项目中的调用过程分析

> * 在openstack 的项目中需要 oslo.messaging 插件包
> * oslo.messaging 的依赖包 kombu, pika.

<!-- more -->

在这里我们只分析 rabbitmq 相关的内容
OpenStack RPC 通信
Openstack 组件内部的 RPC（Remote Producer Call）机制的实现是基于 AMQP(Advanced Message Queuing Protocol)作为通讯模型，从而满足组件内部的松耦合性。AMQP 是用于异步消息通讯的消息中间件协议，AMQP 模型有四个重要的角色:

> * Exchange：根据 Routing key 转发消息到对应的 Message Queue 中
> * Routing key：用于 Exchange 判断哪些消息需要发送对应的 Message Queue
> * Publisher：消息发送者，将消息发送的 Exchange 并指明 Routing Key，以便 Message Queue 可以正确的收到消息
> * Consumer：消息接受者，从 Message Queue 获取消息

消息发布者 Publisher 将 Message 发送给 Exchange 并且说明 Routing Key。Exchange 负责根据 Message 的 Routing Key 进行路由，将 Message 正确地转发给相应的 Message Queue。监听在 Message Queue 上的 Consumer 将会从 Queue 中读取消息。
Routing Key 是 Exchange 转发信息的依据，因此每个消息都有一个 Routing Key 表明可以接受消息的目的地址，而每个 Message Queue 都可以通过将自己想要接收的 Routing Key 告诉 Exchange 进行 binding，这样 Exchange 就可以将消息正确地转发给相应的 Message Queue。图 2 就是 AMQP 消息模型。
图 2. AMQP 消息模型
![AMQP](https://www.ibm.com/developerworks/cn/cloud/library/1403_renmm_opestackrpc/image005.gif)

AMQP 定义了三种类型的 Exchange，不同类型 Exchange 实现不同的 routing 算法：

> * Direct Exchange：Point-to-Point 消息模式，消息点对点的通信模式，Direct Exchange 根据 Routing Key 进行精确匹配，只有对应的 Message Queue 会接受到消息
> * Topic Exchange：Publish-Subscribe(Pub-sub)消息模式，Topic Exchange 根据 Routing Key 进行模式匹配，只要符合模式匹配的 Message Queue 都会收到消息 (模糊匹配)
> * Fanout Exchange：广播消息模式，Fanout Exchange 将消息转发到所有绑定的 Message Queue

## 2， neutron 中所使用的 oslo.messaging 的服务分析

这个用dhcp-agent 服务为例子

```python
# neutron/cmd/eventlet/agents/dhcp.py

from neutron.agent import dhcp_agent

def main():
    dhcp_agent.main()

# neutron/agent/dhcp_agent.py

from neutron.common import config as common_config

def main():
    register_options(cfg.CONF)
    common_config.init(sys.argv[1:])
    config.setup_logging()
    server = neutron_service.Service.create(
        binary='neutron-dhcp-agent',
        topic=topics.DHCP_AGENT,
        report_interval=cfg.CONF.AGENT.report_interval,
        manager='neutron.agent.dhcp.agent.DhcpAgentWithStateReport')
    service.launch(cfg.CONF, server).wait()

# 这里分析 common_config.init(sys.argv[1:])
def init(args, **kwargs):
    cfg.CONF(args=args, project='neutron',
             version='%%(prog)s %s' % version.version_info.release_string(),
             **kwargs)

    # FIXME(ihrachys): if import is put in global, circular import
    # failure occurs
    from neutron.common import rpc as n_rpc
    n_rpc.init(cfg.CONF)

    # Validate that the base_mac is of the correct format
    msg = attributes._validate_regex(cfg.CONF.base_mac,
                                     attributes.MAC_PATTERN)
    if msg:
        msg = _("Base MAC: %s") % msg
        raise Exception(msg)

#  neutron/common/rpc.py
# 这里分析n_rpc.init
TRANSPORT_ALIASES = {
    'neutron.openstack.common.rpc.impl_fake': 'fake',
    'neutron.openstack.common.rpc.impl_qpid': 'qpid',
    'neutron.openstack.common.rpc.impl_kombu': 'rabbit',
    'neutron.openstack.common.rpc.impl_zmq': 'zmq',
    'neutron.rpc.impl_fake': 'fake',
    'neutron.rpc.impl_qpid': 'qpid',
    'neutron.rpc.impl_kombu': 'rabbit',
    'neutron.rpc.impl_zmq': 'zmq',
}

def init(conf):
    global TRANSPORT, NOTIFIER   
    exmods = get_allowed_exmods()
    TRANSPORT = oslo_messaging.get_transport(conf,
                                             allowed_remote_exmods=exmods,
                                             aliases=TRANSPORT_ALIASES)
    serializer = RequestContextSerializer()
    NOTIFIER = oslo_messaging.Notifier(TRANSPORT, serializer=serializer)

# 这里重点分析 oslo_messaging.get_transport() 方法
# oslo.messaging/transport.py 文件
# 
def get_transport(conf, url=None, allowed_remote_exmods=None, aliases=None):
    allowed_remote_exmods = allowed_remote_exmods or []
    # 这里就涉及到 3 个 配置文件的选项
    # transport_url = transport://user:pass@host1:port[,hostN:portN]/virtual_host
    # rpc_backend = rabbitmq
    # control_exchange = oepnstack

    conf.register_opts(_transport_opts)

    if not isinstance(url, TransportURL):
        url = url or conf.transport_url
        parsed = TransportURL.parse(conf, url, aliases)
        if not parsed.transport:
            raise InvalidTransportURL(url, 'No scheme specified in "%s"' % url)
        url = parsed

    kwargs = dict(default_exchange=conf.control_exchange,
                  allowed_remote_exmods=allowed_remote_exmods)

    # url.transport.split('+')[0] = 'rabbit'
    try:
        mgr = driver.DriverManager('oslo.messaging.drivers',
                                   url.transport.split('+')[0],
                                   invoke_on_load=True,
                                   invoke_args=[conf, url],
                                   invoke_kwds=kwargs)
        # 导入rabbitmq的namespace "oslo.messaging.drivers;" 的setup.cfg 相关的driver 类
        # invoke_on_load=True 并且初始化
        # 初始化的参数invoke_args=[conf, url]， invoke_kwds=kwargs
        #oslo.messaging.drivers =
        #       rabbit = oslo_messaging._drivers.impl_rabbit:RabbitDriver

    except RuntimeError as ex:
        raise DriverLoadFailure(url.transport, ex)

    #初始化 Transport 类
    return Transport(mgr.driver)
# 代码中的driver.DriverManager 为初始化 driver类
class RabbitDriver(amqpdriver.AMQPDriverBase):
    """RabbitMQ Driver

    The ``rabbit`` driver is the default driver used in OpenStack's
    integration tests.

    The driver is aliased as ``kombu`` to support upgrading existing
    installations with older settings.  

    """

    def __init__(self, conf, url,
                 default_exchange=None, 
                 allowed_remote_exmods=None):
        opt_group = cfg.OptGroup(name='oslo_messaging_rabbit',
                                 title='RabbitMQ driver options')
        conf.register_group(opt_group)
        conf.register_opts(rabbit_opts, group=opt_group)
        conf.register_opts(rpc_amqp.amqp_opts, group=opt_group)
        conf.register_opts(base.base_opts, group=opt_group)

        self.missing_destination_retry_timeout = (
            conf.oslo_messaging_rabbit.kombu_missing_consumer_retry_timeout)

        self.prefetch_size = (
            conf.oslo_messaging_rabbit.rabbit_qos_prefetch_count)

        connection_pool = pool.ConnectionPool(
            conf, conf.oslo_messaging_rabbit.rpc_conn_pool_size,
            url, Connection)

        super(RabbitDriver, self).__init__(
            conf, url,
            connection_pool,
            default_exchange,
            allowed_remote_exmods
        )


#这里我们分析 Transport 类的初始化
class Transport(object):

    """A messaging transport.

    This is a mostly opaque handle for an underlying messaging transport
    driver.

    It has a single 'conf' property which is the cfg.ConfigOpts instance used
    to construct the transport object.
    """

    def __init__(self, driver):
        self.conf = driver.conf
        self._driver = driver

#到这里我们的rpc 相关的都初始化完成了

```

# 分析rpc 服务的代码流程 

## 1, rpc 服务的 server 端 也就是 receive 端接受消息

> * rpc server 端就是一个 接受 消息端
> * rpc server 端需要一个 回调方法 collback 方法
> * rpc server 端是创建 consumer (消费者)

### 分析 oslo.messaging 包的分析 oslo_messaging/_drivers/./impl_rabbit.py

```python
    # 创建一个 direct 队列
    # 队列 routing_key = queue_name = topic
    def declare_direct_consumer(self, topic, callback):
        """Create a 'direct' queue.
        In nova's use, this is generally a msg_id queue used for
        responses for call/multicall
        """

        consumer = Consumer(exchange_name=topic,
                            queue_name=topic,
                            routing_key=topic,
                            type='direct',
                            durable=False,
                            exchange_auto_delete=True,
                            queue_auto_delete=False,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues,
                            rabbit_queue_ttl=self.rabbit_transient_queues_ttl)

        self.declare_consumer(consumer)
    
       def declare_topic_consumer(self, exchange_name, topic, callback=None,
                               queue_name=None):
        """Create a 'topic' consumer."""
        consumer = Consumer(exchange_name=exchange_name,
                            queue_name=queue_name or topic,
                            routing_key=topic,
                            type='topic',
                            durable=self.amqp_durable_queues,
                            exchange_auto_delete=self.amqp_auto_delete,
                            queue_auto_delete=self.amqp_auto_delete,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues)

        self.declare_consumer(consumer)
    
    #创建一个 fanout 的 consumer 
    #第一步 需要 验证 exchange 是否存在
    self.exchange = kombu.entity.Exchange(
            name=exchange_name,
            type=type,
            durable=self.durable,
            auto_delete=self.exchange_auto_delete)
    # 第二步 创建 queue  指定 exchange 并且给定 routing_key
    def declare(self, conn):
        """Re-declare the queue after a rabbit (re)connect."""
        self.queue = kombu.entity.Queue(
            name=self.queue_name,
            channel=conn.channel,
            exchange=self.exchange,
            durable=self.durable,
            auto_delete=self.queue_auto_delete,
            routing_key=self.routing_key,
            queue_arguments=self.queue_arguments)

        try:
            LOG.trace('ConsumerBase.declare: '
                      'queue %s', self.queue_name)
            self.queue.declare()
        except conn.connection.channel_errors as exc:
            # NOTE(jrosenboom): This exception may be triggered by a race
            # condition. Simply retrying will solve the error most of the time
            # and should work well enough as a workaround until the race
            # condition itself can be fixed.
            # See https://bugs.launchpad.net/neutron/+bug/1318721 for details.
            if exc.code == 404:
                self.queue.declare()
            else:
                raise

    #在创建 fanout 的消费者时候 不需要指定 routing_key 
    def declare_fanout_consumer(self, topic, callback):
        """Create a 'fanout' consumer."""

        unique = uuid.uuid4().hex
        exchange_name = '%s_fanout' % topic
        queue_name = '%s_fanout_%s' % (topic, unique)

        consumer = Consumer(exchange_name=exchange_name,
                            queue_name=queue_name,
                            routing_key=topic,
                            type='fanout',
                            durable=False,
                            exchange_auto_delete=True,
                            queue_auto_delete=False,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues,
                            rabbit_queue_ttl=self.rabbit_transient_queues_ttl)

        self.declare_consumer(consumer)
  
```

### RPC 发送请求

Client 端发送 RPC 请求由 publisher 发送消息并声明消息地址，consumer 接收消息并进行消息处理，如果需要消息应答则返回处理请求的结果消息。OpenStack RPC 模块提供了 :

> rpc.call，
> rpc.cast,
> rpc.fanout_cast,

三种 RPC 调用方法，发送和接收 RPC 请求。

rpc.call 发送 RPC 请求并返回请求处理结果，

请求处理流程如图 5 所示，
![call](https://www.ibm.com/developerworks/cn/cloud/library/1403_renmm_opestackrpc/image011.jpg)
由 Topic Publisher 发送消息，
Topic Exchange 根据消息地址进行消息转发至对应的 Message Queue 中，
Topic Consumer 监听 Message Queue，发现需要处理的消息则进行消息处理，
并由 Direct Publisher 将请求处理结果消息，请求发送方创建 Direct Consumer 监听消息的返回结果

rpc.cast 发送 RPC 请求无返回，请求处理流程如图 6 所示，与 rpc.call 不同之处在于，不需要请求处理结果的返回，因此没有 Direct Publisher 和 Direct Consumer 处理。
![cast](https://www.ibm.com/developerworks/cn/cloud/library/1403_renmm_opestackrpc/image013.jpg)

图 7. RPC.fanout 消息处理
![fanout_cast](https://www.ibm.com/developerworks/cn/cloud/library/1403_renmm_opestackrpc/image015.jpg)

### cast, call 代码分析

```python
# cast 没有返回结果
 def cast(self, ctxt, method, **kwargs):
        """Invoke a method and return immediately. See RPCClient.cast()."""
        msg = self._make_message(ctxt, method, kwargs)
        ctxt = self.serializer.serialize_context(ctxt)

        if self.version_cap:
            self._check_version_cap(msg.get('version'))
        try:
            self.transport._send(self.target, ctxt, msg, retry=self.retry)
        except driver_base.TransportDriverError as ex:
            raise ClientSendError(self.target, ex)

# call 有返回结果
def call(self, ctxt, method, **kwargs):
        """Invoke a method and wait for a reply. See RPCClient.call()."""
        if self.target.fanout:
            raise exceptions.InvalidTarget('A call cannot be used with fanout',
                                           self.target)

        msg = self._make_message(ctxt, method, kwargs)
        msg_ctxt = self.serializer.serialize_context(ctxt)

        timeout = self.timeout
        if self.timeout is None:
            timeout = self.conf.rpc_response_timeout

        if self.version_cap:
            self._check_version_cap(msg.get('version'))

        try:
            result = self.transport._send(self.target, msg_ctxt, msg,
                                          wait_for_reply=True, timeout=timeout,
                                          retry=self.retry)
        except driver_base.TransportDriverError as ex:
            raise ClientSendError(self.target, ex)
        return self.serializer.deserialize_entity(ctxt, result)

```