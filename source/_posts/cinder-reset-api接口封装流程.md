---
title: cinder reset api接口封装流程
date: 2015-06-17 13:15:33
tags: cinder
categories: Openstack
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# Cinder接口到处理函数的流程

分析 当发送一个 api 请求的时候调用方法的流程:
一个接口 到其 处理 函数的流程:

eg : curl -g -i -X GET http://192.168.122.166:8776/v1/d32d063b07414d9099a1b176e4898d2b/os-quota-sets/d32d063b07414d9099a1b176e4898d2b?usage=False

cinder.api.contrib.quotas.QuotaSetsController.show

<!-- more -->

根据 api-paste.ini 文件知道：

request_id faultwrap sizelimit authtoken keystonecontext apiv1

v1 api的封装流程--> request_id(faultwrap(sizelimit(authtoken(keystonecontext(apiv1)))))

当api服务接受到 请求时候 调用的 request_id faultwrap sizelimit authtoken keystonecontext 各个类的__call__ 方法:

```python
       1, cinder.openstack.common.middleware.request_id:RequestIdMiddleware

            #@ 调用其父类的__call__ 方法:

            cinder.openstack.common.middleware.request_id:RequestIdMiddleware

            def __call__(self, req):

                #@ req = Request: GET /v1/d32d063b07414d9099a1b176e4898d2b/os-quota-sets/d32d063b07414d9099a1b176e4898d2b?usage=False HTTP/1.0

                #@ Accept: application/json

                #@ Accept-Encoding: gzip, deflate, sdch

                #@ Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.6,en;q=0.4

                #@ Cache-Control: no-cache

                #@ Connection: keep-alive

                #@ Content-Type: text/plain

                #@ Csp: active

                #@ Host: 192.168.122.166:8776

                #@ User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/43.0.2357.81 Safari/537.36

                #@ X-Auth-Token: 0ede531e8d774ebcb7689f20bef0caca

                response = self.process_request(req)

                #@ 该函数功能就是 设置请求req 的id 然后加入到请求的变量中

                #@ def process_request(self, req):

                #@    self.req_id = context.generate_request_id()

                      self.req_id = str: req-8e464c2e-90b2-4b28-b321-1e3486dde93a

                        #@cinder.openstack.common.context

                        #@ def generate_request_id():

                        #@     return 'req-%s' % uuidutils.generate_uuid()

                                    #@ str(uuid.uuid4())

                                #@ str: req-8e464c2e-90b2-4b28-b321-1e3486dde93a

                #@    req.environ[ENV_REQUEST_ID] = self.req_id

                      req.environ[ENV_REQUEST_ID]  = req-8e464c2e-90b2-4b28-b321-1e3486dde93a

                if response:

                    #@ response = None

                    return response

                response = req.get_response(self.application)

                    #@ req.get_response 会调用cinder.api.middleware.fault:FaultWrapper 类的 __call__ 方法

                        #@ 分析 会调用cinder.api.middleware.fault:FaultWrapper.__call__ 方法

                        def __call__(self, req):

                            try:

                                return req.get_response(self.application)

                                    #@ 分析 会调用 cinder.api.middleware.sizelimit:RequestBodySizeLimiter.__call__ 方法

                                        def __call__(self, req):

                                            #@ req.content_length = None 

                                            if req.content_length > CONF.osapi_max_request_body_size:

                                                #@ 判断 请求的长度是否大于 配置文件中的 osapi_max_request_body_size= 114688

                                                msg = _("Request is too large.")

                                                raise webob.exc.HTTPRequestEntityTooLarge(explanation=msg)

                                            if req.content_length is None and req.is_body_readable:

                                                limiter = LimitingReader(req.body_file,

                                                                 CONF.osapi_max_request_body_size)

                                            req.body_file = limiter

                                            return self.application

                                                #@ 分析 cinder.api.middleware.sizelimit:RequestBodySizeLimiter 类的作用是在 检查 req bady 的长度

                                                #@ 分析 会调用keystoneclient.middleware.auth_token:filter_factory.__call__ 方法

                                                def __call__(self, env, start_response):

                                                    self.LOG.debug('Authenticating user token')

                                                    #@ env = dict: {'HTTP_CSP': 'active', 'SCRIPT_NAME': '/v1', 'webob.adhoc_attrs': {'response': }, 'REQUEST_METHOD': 'GET', 'PATH_INFO': '/d32d063b07414d9099a1b176e4898d2b/os-quota-sets/d32d063b07414d9099a1b176e4898d2b', 'SERVER_PROTOCOL': 'HTTP/1.0', 'QUERY_STRING': 'usage=False', 'HTTP_X_AUTH_TOKEN': '0ede531e8d774ebcb7689f20bef0caca', 'HTTP_USER_AGENT': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/43.0.2357.81 Safari/537.36', 'HTTP_CONNECTION': 'keep-alive', 'REMOTE_PORT': '32828', 'SERVER_NAME': '192.168.122.166', 'REMOTE_ADDR': '192.168.122.1', 'eventlet.input': , 'wsgi.url_scheme': 'http', 'SERVER_PORT': '8776', 'wsgi.input': , 'HTTP_HOST': '192.168.122.166:8776', 'wsgi.multithread': True, 'HTTP_CACHE_CONTROL': 'no-cache', 'eventlet.posthooks': [], 'HTTP_ACCEPT': 'application/json', 'openstack.request_id': 'req-78794fde-a219-40be-a181-d416d8a4d29e...

                                                    self._token_cache.initialize(env)

                                                    try:

                                                        self._remove_auth_headers(env)

                                                        user_token = self._get_user_token_from_header(env)

                                                        token_info = self._validate_user_token(user_token, env)

                                                        #@ 验证token 方法

                                                        env['keystone.token_info'] = token_info

                                                        user_headers = self._build_user_headers(token_info)

                                                        self._add_headers(env, user_headers)

                                                        return self.app(env, start_response)

                                                    except InvalidUserToken:

                                                        if self.delay_auth_decision:

                                                            self.LOG.info(

                                                                'Invalid user token - deferring reject downstream')

                                                            self._add_headers(env, {'X-Identity-Status': 'Invalid'})

                                                            return self.app(env, start_response)

                                                        else:

                                                            self.LOG.info('Invalid user token - rejecting request')

                                                            return self._reject_request(env, start_response)

                                                    except ServiceError as e:

                                                        self.LOG.critical('Unable to obtain admin token: %s', e)

                                                        resp = MiniResp('Service unavailable', env)

                                                        start_response('503 Service Unavailable', resp.headers)

                                                        return resp.body

                                                #@ 分析 keystoneclient.middleware.auth_token:filter_factory 验证 token

                                                #@ 分析 cinder.api.middleware.auth:CinderKeystoneContext.__call__ 方法

                                                        @webob.dec.wsgify(RequestClass=base_wsgi.Request)

                                                        def __call__(self, req):

                                                            user_id = req.headers.get('X_USER')

                                                            user_id = req.headers.get('X_USER_ID', user_id)

                                                            if user_id is None:

                                                                LOG.debug("Neither X_USER_ID nor X_USER found in request")

                                                                return webob.exc.HTTPUnauthorized()

                                                                # get the roles

                                                                roles = [r.strip() for r in req.headers.get('X_ROLE', '').split(',')]

                                                                if 'X_TENANT_ID' in req.headers:

                                                                    # This is the new header since Keystone went to ID/Name

                                                                    project_id = req.headers['X_TENANT_ID']

                                                                else:

                                                                    # This is for legacy compatibility

                                                                    project_id = req.headers['X_TENANT']

                                                                project_name = req.headers.get('X_TENANT_NAME')

                                                                req_id = req.environ.get(request_id.ENV_REQUEST_ID)

                                                                # Get the auth token

                                                                auth_token = req.headers.get('X_AUTH_TOKEN',

                                                                                             req.headers.get('X_STORAGE_TOKEN'))

                                                                # Build a context, including the auth_token...

                                                                remote_address = req.remote_addr

    

                                                                service_catalog = None

                                                                if req.headers.get('X_SERVICE_CATALOG') is not None:

                                                                    try:

                                                                        catalog_header = req.headers.get('X_SERVICE_CATALOG')

                                                                        service_catalog = jsonutils.loads(catalog_header)

                                                                    except ValueError:

                                                                        raise webob.exc.HTTPInternalServerError(

                                                                            _('Invalid service catalog json.'))

                                                                if CONF.use_forwarded_for:

                                                                    remote_address = req.headers.get('X-Forwarded-For', remote_address)

                                                                ctx = context.RequestContext(user_id,

                                                                                             project_id,

                                                                                             project_name=project_name,

                                                                                             roles=roles,

                                                                                             auth_token=auth_token,

                                                                                             remote_address=remote_address,

                                                                                             service_catalog=service_catalog,

                                                                                             request_id=req_id)

                                                                req.environ['cinder.context'] = ctx

                                                                return self.application 

                                                        #@ cinder.api.middleware.auth:CinderKeystoneContext.__call__ 作用是 创建Context 上下文 封装 req 请求

                                                        

                                                        #@ 接着会调用 cinder.api.openstack.wsgi.Resource 类的__call__ 方法

                                                        #@ Resource.__call__ ---> Resource._process_stack ---> Resource.dispatch 调用流程

                                                        #@  def dispatch(self, method, request, action_args):

                                                        #@        """Dispatch a call to the action-specific method."""

                                                        #@       return method(req=request, **action_args)

                                                        #@      method = instancemethod: >

                                                        #@  分析 method 方法则可知 dispatch 会映射到 cinder.api.contrib.quotas.QuotaSetsController.show 方法

                           except Exception as ex:

                                return self._error(ex, req)

                        #@ 分析 会调用cinder.api.middleware.fault:FaultWrapper 作用在 异常处理

                return self.process_response(response)
```