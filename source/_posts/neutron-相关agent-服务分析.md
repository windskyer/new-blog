---
title: neutron 相关agent 服务分析
categories: Openstack
sage: false
date: 2016-04-19 18:15:51
tags: neutron
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# neutron 相关agent 服务分析

我们回到 dhcp_agent 服务在来看代码:
<!-- more -->

```python
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

# 接着来分析 attributes._validate_regex(cfg.CONF.base_mac,
#                                    attributes.MAC_PATTERN)
通过re 模块来判断 mac 地址是否合格
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

# 接着分析 server = neutron_service.Service.create() 

    @classmethod
    def create(cls, host=None, binary=None, topic=None, manager=None,
               report_interval=None, periodic_interval=None,
               periodic_fuzzy_delay=None):
        """Instantiates class and passes back application object.

        :param host: defaults to CONF.host
        :param binary: defaults to basename of executable
        :param topic: defaults to bin_name - 'neutron-' part
        :param manager: defaults to CONF.<topic>_manager
        :param report_interval: defaults to CONF.report_interval
        :param periodic_interval: defaults to CONF.periodic_interval
        :param periodic_fuzzy_delay: defaults to CONF.periodic_fuzzy_delay

        """
        if not host:
            host = CONF.host
        if not binary:
            binary = os.path.basename(inspect.stack()[-1][1])
        if not topic:
            topic = binary.rpartition('neutron-')[2]
            topic = topic.replace("-", "_")
        if not manager:
            # manager = neutron.agent.dhcp.agent.DhcpAgentWithStateReport
            manager = CONF.get('%s_manager' % topic, None)
        if report_interval is None:
            report_interval = CONF.report_interval
        if periodic_interval is None:
            periodic_interval = CONF.periodic_interval
        if periodic_fuzzy_delay is None:
            periodic_fuzzy_delay = CONF.periodic_fuzzy_delay
        service_obj = cls(host, binary, topic, manager,
                          report_interval=report_interval,
                          periodic_interval=periodic_interval,
                          periodic_fuzzy_delay=periodic_fuzzy_delay)

        return service_obj
```

##初始化manager 服务的详细分析

> manager =  neutron.agent.dhcp.agent.DhcpAgentWithStateReport 的详细分析
```
class DhcpAgentWithStateReport(DhcpAgent):               
    def __init__(self, host=None, conf=None):
        super(DhcpAgentWithStateReport, self).__init__(host=host, conf=conf)
        self.state_rpc = agent_rpc.PluginReportStateAPI(topics.PLUGIN)
        #获取 rpc 服务器
        self.agent_state = {
            'binary': 'neutron-dhcp-agent',
            'host': host,
            'topic': topics.DHCP_AGENT, 
            'configurations': {         
                'dhcp_driver': self.conf.dhcp_driver,
                'use_namespaces': self.conf.use_namespaces,
                'dhcp_lease_duration': self.conf.dhcp_lease_duration,
                'log_agent_heartbeats': self.conf.AGENT.log_agent_heartbeats},
            'start_flag': True,         
            'agent_type': constants.AGENT_TYPE_DHCP}
        #获取 rpc 更新的状态值 agent_state
        report_interval = self.conf.AGENT.report_interval
        self.use_call = True
        #循环执行每隔 report_interval 秒
        if report_interval:
            self.heartbeat = loopingcall.FixedIntervalLoopingCall(
                self._report_state)
            self.heartbeat.start(interval=report_interval)
    
    #状态更新
    def _report_state(self):
        try:
            self.agent_state.get('configurations').update(
                self.cache.get_state())
            ctx = context.get_admin_context_without_session()
            self.state_rpc.report_state(ctx, self.agent_state, self.use_call)
            self.use_call = False
        except AttributeError:
            # This means the server does not support report_state
            LOG.warn(_LW("Neutron server does not support state report."
                         " State report for this agent will be disabled."))
            self.heartbeat.stop()
            self.run()
            return
        except Exception:
            LOG.exception(_LE("Failed reporting state!"))
            return
        if self.agent_state.pop('start_flag', None):
            self.run()

```

##dhcp 服务所要执行的linux的shell 命令汇总
> * **kill -9 $pid** 关闭dnsmasq 服务

> * **/var/lib/neutron/dhcp/827$uuid** 删除dnsmasq 的配置文件

> * **u'ip netns exec qdhcp-827b355e-4348-41e8-96c7-c99e0ba08126 ip link set ns-b9f61118-2f up'** 启动网卡

> * **['sudo', 'ip', 'netns', 'exec', 'qdhcp-827b355e-4348-41e8-96c7-c99e0ba08126', 'ip', '-o', 'link', 'show', 'ns-b9f61118-2f']**

> * **sudo ip netns exec qdhcp-827b355e-4348-41e8-96c7-c99e0ba08126 ip addr show ns-b9f61118-2f permanent**

> * **sudo ip netns exec qdhcp-827b355e-4348-41e8-96c7-c99e0ba08126 ip route list dev ns-b9f61118-2f** 

> * **sudo ip netns exec qdhcp-827b355e-4348-41e8-96c7-c99e0ba08126 dnsmasq --no-hosts --no-resolv --strict-order --except-interface=lo --pid-file=/var/lib/neutron/dhcp/827b355e-4348-41e8-96c7-c99e0ba08126/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/827b355e-4348-41e8-96c7-c99e0ba08126/host --addn-hosts=/var/lib/neutron/dhcp/827b355e-4348-41e8-96c7-c99e0ba08126/addn_hosts --dhcp-optsfile=/var/lib/neutron/dhcp/827b355e-4348-41e8-96c7-c99e0ba08126/opts --dhcp-leasefile=/var/
lib/neutron/dhcp/827b355e-4348-41e8-96c7-c99e0ba08126/leases --dhcp-match=set:ipxe,175 --bind-interfaces --interface=ns-b9f61118-2f --dhcp-range=set:tag0,192.168.222.0,static,86400s --dhcp-lease-max=256 --conf-file= --domain=openstacklocal** *启动 dnsmasq 服务* 
*总结 每一个namespace网络都会有相应的 dnsmasq 服务*


##linux bridge 服务所要执行的linux的shell命令汇总

> * ip link add vxlan-1 type vlan id 1 dev eth0 proxy
> * bridge fdb append 00:00:00:00:00:00 dev vxlan-1 dst 1.1.1.1
> * ip -o link show vxlan-1
> * sudo ip link set vxlan-1 down
> * sudo ip link delete vxlan-1 





