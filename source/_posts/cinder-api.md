---
title: cinder-api
date: 2015-06-17 13:17:30
tags: cinder
categories: Openstack
---

# cinder api服务启动子进程分析

<!-- more -->

```python
#!/usr/bin/evn python

#author leidong

#@ 依赖包的分析

#@ webob 模块 简单的说，WebOb是一个用于对WSGI request环境进行包装（也就是变得易用）以及用于创建WSGI response的一个包。

#@ PasteDeploy生成WSGI的Application 就是url 对应的是那些类, 简单来说就是  Application 中的各种url对应的类 使用了 webob 和 Routes 这两个模块 

#@ cinder-api 服务的启动 主要包括 主进程 和 子进程

#@ 主进程 功能分析和 start-volume.py 分析一样

#@ 以下分析的是 子进程

if __name__ == '__main__':

    CONF(sys.argv[1:], project='cinder',

         version=version.version_string())

    logging.setup("cinder")

    utils.monkey_patch()

        #@ monkey_pathch 调用的是 __import__(module)

        #@ monkey_patch_modules = cinder.volume.volume_types:paxes_cinder.volume.volume_type_decorator

        #@ cinder.volume.volume_types 此类 是  __import__('cinder.volume.volume_types') 导入是一个模块 cinder.volume.volume_types.py

        #@ paxes_cinder.volume.volume_type_decorator 做为装饰类 importutils.import_class(paxes_cinder.volume.volume_type_decorator) 导入的是 一个类 或者是 一个 函数

        #@ module_data = pyclbr.readmodule_ex(module) 返回字典形式的 模块中的 所有 class 和 function

        #@ {'A': ,

        #@ 'B': ,

        #@  'f': }

        #@ for method, func in inspect.getmembers(clz, inspect.ismethod):

        #@            setattr(

        #@               clz, method,

        #@               decorator("%s.%s.%s" % (module, key, method), func))

        #@ 此函数的功能是 给 指定模块所有方法列出来  在根据 decorator 装饰类的 指定给定的模块那个方法进行 重新设置属性值setattr

    rpc.init(CONF)

    launcher = service.process_launcher()

    server = service.WSGIService('osapi_volume')

    launcher.launch_service(server, workers=server.workers or 1)

    launcher.wait()

#@ 主要分析的是以下代码

    server = service.WSGIService('osapi_volume')

    #@ 初始化 cinder/service:WSGIService 类

    def __init__(self, name, loader=None):

        """Initialize, but do not start the WSGI server.

        :param name: The name of the WSGI server given to the loader.

        :param loader: Loads the WSGI application using the given name.

        :returns: None

        """

        self.name = name

        #@ self.name = 'osapi_volume'

        self.manager = self._get_manager()

        #@

        #@ 

            fl = '%s_manager' % self.name

            #@ f1 = osapi_volume_manager

            if fl not in CONF:

            #@ 在 配置文件中 没有 osapi_volume_manager 选项

                return None

            manager_class_name = CONF.get(fl, None)

            #@ 如果有则获取 api 的管理类

            if not manager_class_name:

            #@ 有选项没有指定管理类

                return None

            manager_class = importutils.import_class(manager_class_name)

            #@ 导入api 的管理类

            return manager_class()

 

        self.loader = loader or wsgi.Loader()

        self.app = self.loader.load_app(name)

        #@ 主要是解析 api-paste.ini 文件

        #@ /v1: openstack_volume_api_v1

        #@       

                [composite:openstack_volume_api_v1]

                use = call:cinder.api.middleware.auth:pipeline_factory

                #@ 调用的是 :pipeline_factory 方法

                #@  pipeline = local_conf[CONF.auth_strategy]

                    #@ 获取的是 api-paste.ini 文件中的pipline = request_id faultwrap sizelimit authtoken keystonecontext apiv1 

                    if not CONF.api_rate_limit:

                        limit_name = CONF.auth_strategy + '_nolimit'

                        pipeline = local_conf.get(limit_name, pipeline)

                        #@ 获取的是 api-paste.ini 文件中的keystone_nolimit = request_id faultwrap sizelimit authtoken keystonecontext apiv1

                    pipeline = pipeline.split()

                    filters = [loader.get_filter(n) for n in pipeline[:-1]]

                    #@ 获取列表 

                    #@ request_id ： cinder.openstack.common.middleware.request_id:RequestIdMiddleware.factory 对应方法

                    #@ faultwrap ： cinder.api.middleware.fault:FaultWrapper.factory 对应方法

                    #@ sizelimit : cinder.api.middleware.sizelimit:RequestBodySizeLimiter.factory 对应方法

                    #@ authtoken : keystoneclient.middleware.auth_token:filter_factory

                    #@ keystonecontext : cinder.api.middleware.auth:CinderKeystoneContext.factory

                    app = loader.get_app(pipeline[-1])

                    #@ 获取 v1 ： cinder.api.v1.router:APIRouter.factory

                    filters.reverse()

                    #@ 反转 filters 列表

                    for filter in filters:

                        app = filter(app)

                        #@ 装饰的顺序为

                        #@ request_id(faultwrap(sizelimit(authtoken(keystonecontext(v1)))))

                    return app #@ app = request_id(faultwrap(sizelimit(authtoken(keystonecontext(v1)))))

                    #@ app v1 初始化的过程:

                        v1=cinder.api.v1.router:APIRouter.factory

                        #@ 调用 父类(cinder.api.openstack.APIRouter) 的 类方法 factory

                        #@  def factory(cls, global_config, **local_config):

                                return cls()

                                #@ 初始化自己 调用 父类的 cinder.api.openstack.APIRouter.__init__ 方法

                                #@  ExtensionManager = extensions.ExtensionManager

                                        #@ cinder.api.extensions.ExtensionManager 类

                                #@  if ext_mgr is None:

                                        if self.ExtensionManager:

                                            ext_mgr = self.ExtensionManager()

                                            #@ 初始化cinder.api.extensions.ExtensionManager 

                                            #@  def __init__(self):

                                                LOG.audit(_('Initializing extension manager.'))

                                                #@ 打印 日志文件的 第二部分

                                                #@ 2015-06-10 02:32:21.251 1038 AUDIT cinder.api.extensions [-] Initializing extension manager.

                                                self.cls_list = CONF.osapi_volume_extension

                                                #@ osapi_volume_extension=cinder.api.contrib.standard_extensions

                                                self.extensions = {}

                                                self._load_extensions()

                                        else:

                                            raise Exception(_("Must specify an ExtensionManager class"))

                                    mapper = ProjectMapper()

                                    self.resources = {}

                                    self._setup_routes(mapper, ext_mgr)

                                    self._setup_ext_routes(mapper, ext_mgr)

                                    self._setup_extensions(ext_mgr)

                                    super(APIRouter, self).__init__(mapper) 

                        

            #@ 最后返回 app

            #@  if not CONF.enable_v1_api:

                    del local_conf['/v1']

                if not CONF.enable_v2_api:

                    del local_conf['/v2']

                return paste.urlmap.urlmap_factory(loader, global_conf, **local_conf)

                #@ 返回一个 dict 

        self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")

        self.port = getattr(CONF, '%s_listen_port' % name, 0)

        self.workers = getattr(CONF, '%s_workers' % name, None)

        if self.workers < 1:

            LOG.warn(_("Value of config option %(name)s_workers must be "

                       "integer greater than 1.  Input value ignored.") %

                     {'name': name})

            # Reset workers to default

            self.workers = None

        self.server = wsgi.Server(name,

                                  self.app,

                                  host=self.host,

                                  port=self.port)
```
