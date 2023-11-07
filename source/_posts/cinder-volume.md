---
title: cinder-volume
date: 2015-06-17 13:07:52
tags: cinder
categories: Openstack
---

# cinder-volume服务主进程分析

<!-- more -->

```python
#!/usr/bin/env python

#author zwei

#@ cinder-volume 为例子

######## cinder-volume 服务启动的 流程  ##############

    CONF(sys.argv[1:], project='cinder',

                  version=version.version_string())

    #@ 初始化 /etc/cinder/cinder.conf 配置文件中 的选项值 如果配置文件中没有则 以 代码中 注册的 值为默认值

    logging.setup("cinder")

    #@ 设置 日志文件的目录 和 和 日志处理的 方式 和 日志 输出的 格式 

    utils.monkey_patch()

    #@ 动态导入 模块

        CONF.monkey_patch:

        #@ 根据 配置文件 中的  monkey_patch 的值 True 和 False

        CONF.monkey_patch_modules

        #@ 如果 CONF.monkey_patch 为True 则 会 导入 CONF.monkey_patch_modules 中的 各种 类  使用 __import__(module)

    launcher = service.get_launcher()

    #@ 使用 cindr/service 中的 get_launcher()

    if os.name == 'nt':

        #@ 判断 操作系统类型 nt 表示 windows

        return Launcher()

            #@class Launcher(object):

            #@  def __init__(self):

            #@  self.launch_service = serve

            #@  self.wait = wait

        #@ 表示初始话一个新的类  没有 任何 功能方法 所有 cinder-volume 不能在 windows 上运行 其他服务 类似

    else:

        

        return process_launcher()

        #@  return service.ProcessLauncher()

        #@ 会调用 cinder/openstack/common/service.py 中的ProcessLauncher 类  初始化 该 类

        #@ 初始化 process_launcher 类

        self.children = {}

        self.sigcaught = None

        self.running = True

        self.wait_interval = wait_interval = 0.01 #@ 默认值

        rfd, self.writepipe = os.pipe()

        #@ 返回一个 rfd read 文件描述符 self.writepipe write 文件描述符

        self.readpipe = eventlet.greenio.GreenPipe(rfd, 'r')

        self.handle_signal()

            #@ def handle_signal(self):

            #@      _set_signals_handler(self._handle_signal)

            #@ 

                def _set_signals_handler(handler):

                    signal.signal(signal.SIGTERM, handler)

                    #@ 表示 当程序 收到 signal.SIGTERM 信号的时候 会调用 _handle_signal 函数

                    #@ signal.SIGTERM  参考 kill -l  和 man 7 signal 查看详细信息

                    #@ SIGTERM      15       Term    Termination signal

                    #@ 程序结束(terminate)信号, 与SIGKILL不同的是该信号可以被阻塞和处理。通常用来要求程序自己正常退出，shell命令kill缺省产生这个信号。如果进程终止不了，我们才会尝试SIGKILL

                    #@ 当程序自己 正常结束所 收到的信号

                    signal.signal(signal.SIGINT, handler)

                    #@ 当后台作业要从用户终端读数据时, 该作业中的所有进程会收到SIGTTIN信号. 缺省时这些进程会停止执行. 

                    if _sighup_supported():

                        #@ return hasattr(signal, 'SIGHUP')

                        #@ 查看 signal 是否 有 SIGHUP 属性

                        signal.signal(signal.SIGHUP, handler)

                        #@ 本信号在用户终端连接(正常或非正常)结束时发出, 通常是在终端的控制进程结束时, 通知同一session内的各个作业, 这时它们与控制终端不再关联。

                    #@ http://blog.chinaunix.net/uid-26111972-id-3794962.html 参考文档

            #@ 会调用

            def _handle_signal(self, signo, frame):

                self.sigcaught = signo

                self.running = False

    for backend in CONF.enabled_backends:

        #@获取 配置文件中的 enabled_backends 选项 eg ['vg1','vg2','nfs1','nfs2']

        host = "%s@%s" % (CONF.host, backend)

        #eg: ivm140@nfs1

        server = service.Service.create(host=host,

                                        service_name=backend,

                                        binary='cinder-volume')

        #@ 调用cinder/service.py:service 类的 create 类方法:

               def create(cls, host=None, binary=None, topic=None, manager=None,

               report_interval=None, periodic_interval=None,

               periodic_fuzzy_delay=None, service_name=None):

                

                     if not host:

                        host = CONF.host 

                        #@ 在 不使用enabled_backends 选项时候会去调用 配置文件中的  host 选项 默认值 是eg default=socket.gethostname() 当前系统的主机名称 在 cinder/common/config.py 144 行配置 注册

                     binary = os.path.basename(inspect.stack()[-1][1])

                    #@ 获取当前执行的脚本名称（cinder-volume）

                    subtopic = topic.rpartition('cinder-')[2]

                    #@ 将字符串转换为 元组 ('','cinder-','volume')

                    manager = CONF.get('%s_manager' % subtopic, None)

                    #@ 获取 卷的 管理器 的 类 volume_manager = cinder.volume.manager.VolumeManager

                    report_interval = CONF.report_interval

                    #@ 报告 cinder-volume 服务的状态信息

                    periodic_interval = CONF.periodic_interval

                    periodic_fuzzy_delay = CONF.periodic_fuzzy_delay

                    service_obj = cls(host, binary, topic, manager,

                                                                report_interval=report_interval,

                                                                periodic_interval=periodic_interval,

                                                                periodic_fuzzy_delay=periodic_fuzzy_delay,

                                                                service_name=service_name)

                    #@ 初始化 cinder/service/Service 类 

                    def __init__(self, host, binary, topic, manager, report_interval=None,

                                                  periodic_interval=None, periodic_fuzzy_delay=None,

                                                  service_name=None, *args, **kwargs):

                                super(Service, self).__init__()

                                #@ 初始化 父类

                                #@ 

                                    def __init__(self, threads=1000):

                                        self.tg = threadgroup.ThreadGroup(threads)

                                        #@ 设置协程 池的大小 初始化ThreadGroup 类 

                                        #@ cinder/openstack/common/threadgroup.py:ThreadGroup

                                        #@ 

                                             def __init__(self, thread_pool_size=10):

                                                self.pool = greenpool.GreenPool(thread_pool_size)

                                                #@ 初始化 greenpool.GreenPool(10) 协程池

                                                self.threads = []

                                                self.timers = []

                                        # signal that the service is done shutting itself down:

                                        self._done = threading.Event()

                                        #@ 初始化 threading 模块中的 Event 类

                                 if not rpc.initialized():

                                        #@ rpc.initialized() 返回值 false

                                     rpc.init(CONF)

                                     

                                     #@ rpc 初始化

                                         def init(conf):

                                            global TRANSPORT, NOTIFIER

                                            exmods = get_allowed_exmods()

                                            TRANSPORT = messaging.get_transport(conf,

                                                                                allowed_remote_exmods=exmods,

                                                                                aliases=TRANSPORT_ALIASES)

                                            #@ 会加载 oslo.messaging-1.4.1.dist-info/entry_points.txt 文件中的 oslo.messaging.drivers

                                            #@ 打印的日志

                                            #@ 服务启动时候 最开始打印的日志

                                            #@ 2015-06-08 23:05:02.697 27346 DEBUG stevedore.extension [-] 

                                            #@ found extension EntryPoint.parse('qpid = oslo.messaging._drivers.impl_qpid:QpidDriver') 

                                            #@ _load_plugins /usr/lib/python2.6/site-packages/stevedore/extension.py:157

                                            serializer = RequestContextSerializer(JsonPayloadSerializer())

                                            NOTIFIER = messaging.Notifier(TRANSPORT, serializer=serializer)

                                            #@ 此段中 会注册 notification_driver 和 notification_topics 两个选项

                                            #@ 使用 notification_driver 选项在 oslo/messaging/notifly/notifier:Notifier,__init__ 125 line

                                            #@ 使用 notification_topics 选项在 oslo/messaging/notifly/notifier:Notifier,__init__ 128 line

                                            #@ 调用以下方法

                                            #@       self._driver_mgr = named.NamedExtensionManager(

                                                                        'oslo.messaging.notify.drivers',

                                                                        names=self._driver_names,

                                                                        invoke_on_load=True,

                                                                        invoke_args=[transport.conf],

                                                                        invoke_kwds={

                                                                        'topics': self._topics,

                                                                        'transport': self.transport,

                                                                        }

                                            #@ 最后调用 pkg_resources.iter_entry_points(namespace) namespace='oslo.messaging.notify.drivers' #可以知道 pkg_resources 会查看 所有的entry_points.txt文件 查找 oslo.messaging.notify.drivers

                                            #@ 打印的日志

                                            #@ 服务启动时候 第二部分打印的日志

                                            #@ 会加载 cinder.egg-info/entry_points.txt 文件中的 oslo.messaging.notify.drivers 

                                            #@ 2015-06-08 23:34:38.060 27346 DEBUG stevedore.extension [-] 

                                            #@ found extension EntryPoint.parse('cinder.openstack.common.notifier.no_op_notifier = oslo.messaging.notify._impl_noop:NoOpDriver') 

                                            #@ _load_plugins /usr/lib/python2.6/site-packages/stevedore/extension.py:157

                                            #@ 会加载 oslo.messaging-1.4.1.dist-info/entry_points.txt 文件中的 oslo.messaging.notify.drivers

                                            #@ 2015-06-08 23:38:28.369 27346 DEBUG stevedore.extension [-] 

                                            #@ found extension EntryPoint.parse('log = oslo.messaging.notify._impl_log:LogDriver') _load_plugins /usr/lib/python2.6/site-packages/stevedore/extension.py:157

                                self.host = host

                                #@str: cinder.flftuu.com@vg1

                                self.binary = binary

                                #@ str: cinder-volume

                                self.topic = topic

                                #@ str: cinder-volume

                                self.manager_class_name = manager

                                #@ str: cinder.volume.manager.VolumeManager

                                manager_class = importutils.import_class(self.manager_class_name)

                                #@ _PeriodicTasksMeta:

                                self.manager = manager_class(host=self.host,

                                     service_name=service_name,

                                     *args, **kwargs)

                                #@ 初始化 cinder.volume.manager.VolumeManager 类

                                self.report_interval = report_interval

                                #@ int: 10  10s 跟新数据库状态

                                self.periodic_interval = periodic_interval

                                #@ int: 60  60s 发送消息60s 一次 到 rabbitmq

                                self.periodic_fuzzy_delay = periodic_fuzzy_delay

                                #@ int: 60

                                self.basic_config_check()

                                #@  if CONF.service_down_time <= self.report_interval:

                                #@ 检测 判断服务down 的时间差 和服务更新时间相比较

                                #@ down 的时间差 一定要大于 服务更新的时间差 

                                self.saved_args, self.saved_kwargs = args, kwargs

                                self.timers = []

            launcher.launch_service(server)

            #@ 调用 cinder/openstack/common/service.py:process_launcher 类

            def launch_service(self, service, workers=1):

                #@ workers=1 表示一个默认的cinder-volume 服务只有一个子进程

                wrap = ServiceWrapper(service, workers)

                    #@初始化  ServiceWrapper 

                    #@  

                    class ServiceWrapper(object):

                        def __init__(self, service, workers):

                            self.service = service

                            self.workers = workers

                            self.children = set()

                            self.forktimes = []

                LOG.info(_('Starting %d workers'), wrap.workers)

                #@ 打印第 三部分日志

                #@ 2015-06-09 01:26:58.982 28852 INFO cinder.openstack.common.service [-] Starting 1 workers

                while self.running and len(wrap.children) < wrap.workers:

                    #@ self.running = True , len(wrap.children) = 0 < wrap.workers = 1 

                     self._start_child(wrap)

                        if len(wrap.forktimes) > wrap.workers:

                            #@ len(wrap.forktimes) = 0 > wrap.workers = 1

                            if time.time() - wrap.forktimes[0] < wrap.workers:

                                #@ 当前时间 与 创建子进程时间 做对比 要大于 1s

                                LOG.info(_('Forking too fast, sleeping'))

                                time.sleep(1)

                            wrap.forktimes.pop(0)           

                        wrap.forktimes.append(time.time())

                        #@ 设置创建子进程的时间

                        pid = os.fork()

                        #@ 创建子进程

                        if pid == 0:

                        #@ pid == 0 表示 以下代码在 子进程中运行

                            launcher = self._child_process(wrap.service)

                            while True:

                                self._child_process_handle_signal()

                                status, signo = self._child_wait_for_exit_or_signal(launcher)

                                if not _is_sighup_and_daemon(signo):

                                    break

                                launcher.restart()

                            os._exit(status)

                        LOG.info(_('Started child %d'), pid)

                        #@ 在父进程中输出 第四部分日志 

                        #@ 2015-06-09 01:47:10.358 31196 INFO cinder.openstack.common.service [-] Started child 31218

                        wrap.children.add(pid)

                        #@ 保留 子进程的 pid 

                        self.children[pid] = wrap

                        #@ 设置 子进程 对应的 ServiceWrapper 的实例

                        return pid

    else:

        server = service.Service.create(binary='cinder-volume')

        launcher.launch_service(server)

        #@ 与以上相同

    launcher.wait()

    #@ 调用 cinder/openstack/common/service.py:process_launcher 类 的 wait() 方法

        LOG.debug(_('Full set of CONF:'))

        #@ 2015-06-09 01:47:10.360 31196 DEBUG cinder.openstack.common.service [-] Full set of CONF: wait /root/workspace/cinder/cinder/openstack/common/service.py:384

        CONF.log_opt_values(LOG, std_logging.DEBUG)

        #@ 打印主进程的 第五部分日志

        #@ 打印所有的 有关 cinder-volume 服务的 配置文件参数

        #@ 

        #@ logger.log(lvl, "*" * 80)

        #@ logger.log(lvl, "Configuration options gathered from:")

        #@ logger.log(lvl, "command line args: %s", self._args)

        #@ logger.log(lvl, "config files: %s", self.config_file)

        #@ logger.log(lvl, "=" * 80)

        #@ 2015-06-09 01:47:10.367 31196 DEBUG cinder.openstack.common.service [-] allowed_direct_url_schemes     = [] log_opt_values /usr/lib/python2.6/site-packages/oslo_config/cfg.py:2184

        #@ 参数的 排列顺序为 a-z 字母列表

        while True:

            self.handle_signal()

            #@ 设置 信号量处理 函数

            #@ 同上

            self._respawn_children()

            #@ 切换 eventlet 中的 协程 

            #@    

                while self.running:

                    wrap = self._wait_child()

                        def _wait_child(self):

                            try:

                                # Don't block if no child processes have exited

                                pid, status = os.waitpid(0, os.WNOHANG)

                                #@ 获取子进程的 状态信息

                                if not pid:

                                    return None

                            except OSError as exc:

                                if exc.errno not in (errno.EINTR, errno.ECHILD):

                                    raise

                                return None

                            if os.WIFSIGNALED(status):

                                sig = os.WTERMSIG(status)

                                LOG.info(_('Child %(pid)d killed by signal %(sig)d'),

                                        dict(pid=pid, sig=sig))

                            else:

                                code = os.WEXITSTATUS(status)

                                LOG.info(_('Child %(pid)s exited with status %(code)d'),

                                        dict(pid=pid, code=code))

                            if pid not in self.children:

                                LOG.warning(_('pid %d not in child list'), pid)

                                return None

                            wrap = self.children.pop(pid)

                            wrap.children.remove(pid)

                            return wrap

                    if not wrap:

                        # Yield to other threads if no children have exited

                        # Sleep for a short time to avoid excessive CPU usage

                        # (see bug #1095346)

                        eventlet.greenthread.sleep(self.wait_interval)

                        #@ 停止当前的协程， 使其他的协程 继续运行下去 停止的时间为 self.wait_interval 默认为0.01s

                        continue

                    #@ 如果没有信号量 输入到程序 则会一直循环

                    while self.running and len(wrap.children) < wrap.workers: 

                        self._start_child(wrap)

            #@ 当 键盘发送 ctrl + c 时候 调用self.handle_signal 方法  

            #@ 最终会设置这几个变量

            #@ self.sigcaught = signo

            #@ self.running = False

            #@ 然后调用self._respawn_children()  不做处理

            #@ 会调用以下代码

            if self.sigcaught:

                signame = _signo_to_signame(self.sigcaught)

                #@ 获取发送信号量的名称

                LOG.info(_('Caught %s, stopping children'), signame)

                #@ 主进程的 结束是的 日志打印

            if not _is_sighup_and_daemon(self.sigcaught):

                #@ 判断是 信号量 是不是 SIGHUB

                break

                #@ 推出循环

            for pid in self.children:

                os.kill(pid, signal.SIGHUP)

                #@ 杀掉子进程

            self.running = True

            self.sigcaught = None

         for pid in self.children:

            try:

                os.kill(pid, signal.SIGTERM)

                #@ 杀掉子进程

            except OSError as exc:

                if exc.errno != errno.ESRCH:

                    raise

        # Wait for children to die

        if self.children:

            LOG.info(_('Waiting on %d children to exit'), len(self.children))

            #@ 主进程的 结束是的 日志打印

            while self.children:

                self._wait_child()

                    try:

                    # Don't block if no child processes have exited

                        pid, status = os.waitpid(0, os.WNOHANG)

                        #@ 查看子进程的状态信息

                        if not pid:

                            return None

                        except OSError as exc:

                            if exc.errno not in (errno.EINTR, errno.ECHILD):

                                raise

                            return None

                    if os.WIFSIGNALED(status):

                        sig = os.WTERMSIG(status)

                        LOG.info(_('Child %(pid)d killed by signal %(sig)d'),

                                 dict(pid=pid, sig=sig))

                        #@ 主进程的 结束是的 日志打印

                        #@ 打印子进程pid

                    else:

                        code = os.WEXITSTATUS(status)

                        LOG.info(_('Child %(pid)s exited with status %(code)d'),

                                 dict(pid=pid, code=code))

                    if pid not in self.children:

                        LOG.warning(_('pid %d not in child list'), pid)

                        return None

                    wrap = self.children.pop(pid)

                    wrap.children.remove(pid)

                    return wrap

                    #@ 到此主进程所做的 工作 分析结束

#@------------os.fork() 创建子进程  ， 分析 子进程-----------------------------@#

                        if pid == 0:

                        #@ pid == 0 表示 以下代码在 子进程中运行

                            launcher = self._child_process(wrap.service)

                            #@

                            #@

                                def _child_process(self, service):

                                    self._child_process_handle_signal()

                                    eventlet.hubs.use_hub()

                                    #@ 使用 系统的 AIO （异步IO） 使协程 和 线程一样具有异步io调用系统

                                    # Close write to ensure only parent has it open

                                    os.close(self.writepipe)

                                    # Create greenthread to watch for parent to close pipe

                                    eventlet.spawn_n(self._pipe_watcher)

                                    #@ 创建一个协程 来执行 self._pipe_watcher 函数

                                    #@ 

                                    #@ 

                                            def _pipe_watcher(self):

                                                # This will block until the write end is closed when the parent

                                                # dies unexpectedly

                                                self.readpipe.read()

                                                LOG.info(_('Parent process has died unexpectedly, exiting'))

                                                sys.exit(1)

                                    # Reseed random number generator

                                    random.seed()

                                    #@ 设置伪随机种子

                                    launcher = Launcher()

                                    #@ 初始化 Launcher 类

                                    #@ 

                                    #@  self.services = Services()

                                    #@  初始化 Services 类

                                    #@       def __init__(self):

                                                self.services = []

                                                self.tg = threadgroup.ThreadGroup()

                                                #@ 初始化 ThreadGroup 类

                                                self.done = threading.Event()

                                    #@  self.backdoor_port = eventlet_backdoor.initialize_if_enabled()

                                    launcher.launch_service(service)

                                    return launcher

                            while True:

                                self._child_process_handle_signal()

                                status, signo = self._child_wait_for_exit_or_signal(launcher)

                                if not _is_sighup_and_daemon(signo):

                                    break

                                launcher.restart()

                            os._exit(status)
```