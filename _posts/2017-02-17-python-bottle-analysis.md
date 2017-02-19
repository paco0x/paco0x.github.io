---
layout:     post
title:      Python bottle 源码解析
subtitle:   
date:       2017-02-17 19:00:00
author:     Liao
catalog:    true
header-img: img/in-post/python-bottle/bottle.jpg
permalink:  /python-bottle/
tags:
    - Python
    - Web
---

# 简介

Bottle 是一个轻量级的 Python Web 框架，它的代码量只有 4300 行（为了突出其精简性，所有代码放在了一个文件中）。但是麻雀虽小，五脏俱全，Bottle 提供了一个 Web 框架有必须的基本功能，如：

* 路由，支持动态的 URL 路由
* 模板，内置了模板功能，同时支持 mako, jinja2 和 cheetah
* 表单，文件上传，cookies，headers 等的处理
* 支持多种 WSGI Server，如 cherrypy, eventlet, twisted 等等

关于 bottle 的使用，可以参考官方文档：[https://bottlepy.org/docs/dev/index.html](https://bottlepy.org/docs/dev/index.html)。

# 运行

使用 bottle 写一个 hello world 是这样的：

```python
from bottle import route, run, template

@route('/hello/<name>')
def index(name):
    return template('<b>Hello {{name}}</b>!', name=name)

run(host='localhost', port=8080)
```

这里我们可以看到，使用 bottle 作为 Web 框架它为我们提供了如下功能：

* 路由，即将 URL 与处理函数进行绑定，在 bottle 中是使用装饰器实现的
* 模板，动态生成 HTML 响应
* 运行，提供 WSGI Server 运行 Web 服务器

我们首先从运行说起，要了解它是如何运行一个 Web 服务器的，首先需要知道 Python WSGI 的概念，可以参考我的上一篇文章：[Python WSGI 初探](http://liaoph.com/python-wsgi/)。

为了运行一个 Web 服务器，首先需要一个 WSGI 服务器，Python 的自带库 `wsgiref` 就能提供 WSGI 服务器。其次，需要有一个 application 对象，WSGI 服务器收到请求后会调用这个 application 对象来完成处理。

Bottle 中 Web 服务器的运行是通过 `run()` 函数来完成，这里我们并没有指定 application 对象，也没有指定 WSGI 服务器，实际上 bottle 为我们生成了默认的 application 对象和 WSGI 服务器，其 `run()` 函数如下（这里为了简略省去了一些和运行无关的代码）：

```python
def run(app=None,
        server='wsgiref',
        host='127.0.0.1',
        port=8080,
        interval=1,
        reloader=False,
        ...(略)
        ):
    try:
        app = app or default_app()
        if isinstance(app, basestring):
            app = load_app(app)
        if not callable(app):
            raise ValueError("Application is not callable: %r" % app)

        if server in server_names:
            server = server_names.get(server)
        if isinstance(server, basestring):
            server = load(server)
        if isinstance(server, type):
            server = server(host=host, port=port, **kargs)
        if not isinstance(server, ServerAdapter):
            raise ValueError("Unknown or unsupported server: %r" % server)

        server.run(app)
        ...(略)
```

我们可以看到，如果不显式指定，bottle 会生成一个默认的 application 对象：

```python
app = app or default_app()
```

实际上 bottle 内部可以存在多个 application 对象，它将这些 application 对象保存到一个栈中，这个栈是一个 `AppStack` 类对象：

```python
apps = app = default_app = AppStack()
```

`AppStack` 的定义如下：

```python
class AppStack(list):
    """ A stack-like list. Calling it returns the head of the stack. """

    def __call__(self):
        """ Return the current default application. """
        return self.default

    def push(self, value=None):
        """ Add a new :class:`Bottle` instance to the stack """
        if not isinstance(value, Bottle):
            value = Bottle()
        self.append(value)
        return value
    new_app = push

    @property
    def default(self):
        try:
            return self[-1]
        except IndexError:
            return self.push()
```

可以看到 AppStack 中会以栈的形式保存所有 Application，一个 Application 就是一个 `Bottle` 实例。

同时如果不指定 WSGI 服务器，bottle 会动态的加载一个 WSGIServer，加载的代码如下：

```python
    if server in server_names:
        server = server_names.get(server)
    if isinstance(server, basestring):
        server = load(server)
    if isinstance(server, type):
        server = server(host=host, port=port, **kargs)
    if not isinstance(server, ServerAdapter):
        raise ValueError("Unknown or unsupported server: %r" % server)
    ...(省略)
    if reloader:
        lockfile = os.environ.get('BOTTLE_LOCKFILE')
        bgcheck = FileCheckerThread(lockfile, interval)
        with bgcheck:
            server.run(app)
        if bgcheck.status == 'reload':
            sys.exit(3)
    else:
        server.run(app)
```

这里首先从 `server_names` 中获取需要加载的 WSGI Server，`server_name` 是定义好的全局变量，`load()` 函数用于动态加载 WSGI Server 对象，加载完成后，会使用 `server.run()` 方法运行 WSGI Server。`server_names` 这个全局变量保存了所有支持的 WSGI 服务器的映射表：

```python
server_names = {
    'cgi': CGIServer,
    'flup': FlupFCGIServer,
    'wsgiref': WSGIRefServer,
    'waitress': WaitressServer,
    'cherrypy': CherryPyServer,
    'paste': PasteServer,
    'fapws3': FapwsServer,
    'tornado': TornadoServer,
    'gae': AppEngineServer,
    'twisted': TwistedServer,
    'diesel': DieselServer,
    'meinheld': MeinheldServer,
    'gunicorn': GunicornServer,
    'eventlet': EventletServer,
    'gevent': GeventServer,
    'rocket': RocketServer,
    'bjoern': BjoernServer,
    'aiohttp': AiohttpServer,
    'uvloop': AiohttpUVLoopServer,
    'auto': AutoServer,
}
```

这里包含的 bottle 所支持的所有的 WSGI Server，以 `WSGIRefServer` 为例：

```python
class WSGIRefServer(ServerAdapter):
    def run(self, app):  # pragma: no cover
        from wsgiref.simple_server import make_server
        from wsgiref.simple_server import WSGIRequestHandler, WSGIServer
        import socket

        class FixedHandler(WSGIRequestHandler):
            def address_string(self):  # Prevent reverse DNS lookups please.
                return self.client_address[0]

            def log_request(*args, **kw):
                if not self.quiet:
                    return WSGIRequestHandler.log_request(*args, **kw)

        handler_cls = self.options.get('handler_class', FixedHandler)
        server_cls = self.options.get('server_class', WSGIServer)

        if ':' in self.host:  # Fix wsgiref for IPv6 addresses.
            if getattr(server_cls, 'address_family') == socket.AF_INET:

                class server_cls(server_cls):
                    address_family = socket.AF_INET6

        self.srv = make_server(self.host, self.port, app, server_cls,
                               handler_cls)
        self.port = self.srv.server_port  # update port actual port (0 means random)
        try:
            self.srv.serve_forever()
        except KeyboardInterrupt:
            self.srv.server_close()  # Prevent ResourceWarning: unclosed socket
            raise

```

所有的 bottle WSGIServer 都必须是 `ServerAdapter` 的子类，且需要实现一个 `run()` 方法，bottle 在加载 WSGIServer 时会先进行类型检查，然后调用 `run()` 方法启动 WSGIServer。这里的 `WSGIRefServer` 类，内部实际上使用了 `wsgiref` 中提供的 WSGI Server，bottle 做了一些简单的封装和扩展。

# Request & Response

作为一个 Web 框架，需要接受请求，处理请求，生成响应返回给用户，bottle 把请求和响应进行了一些封装，保存在全局变量 `request` 和 `response`（线程安全的）中：

```python
#: A thread-safe instance of :class:`LocalRequest`. If accessed from within a
#: request callback, this instance always refers to the *current* request
#: (even on a multi-threaded server).
request = LocalRequest()

#: A thread-safe instance of :class:`LocalResponse`. It is used to change the
#: HTTP response for the *current* request.
response = LocalResponse()
```

`request` 和 `response` 永远表示「当前」的请求和响应对象，它们是线程安全的。

`LocalRequest` 是 `BaseRequest` 的基类。`BaseRequest` 对 `environ` 字典（即从 WSGI Server 传入的环境变量字典）进行了封装，它可以被作为类似字典的容器来使用，但是所有的键值访问和存储实际上都是通过内部的 `environ` 字典完成的：

```python
class BaseRequest(object):
    __slots__ = ('environ', )
    def __init__(self, environ=None):
        """ Wrap a WSGI environ dictionary. """
        #: The wrapped WSGI environ dictionary. This is the only real attribute.
        #: All other attributes actually are read-only properties.
        self.environ = {} if environ is None else environ
        self.environ['bottle.request'] = self
        
    def __getitem__(self, key):
        return self.environ[key]

    def __delitem__(self, key):
        self[key] = ""
        del (self.environ[key])

    def __iter__(self):
        return iter(self.environ)

    def __len__(self):
        return len(self.environ)
        
    def __setitem__(self, key, value):
        """ Change an environ value and clear all caches that depend on it. """

        if self.environ.get('bottle.request.readonly'):
            raise KeyError('The environ dictionary is read-only.')

        self.environ[key] = value
        todelete = ()

        if key == 'wsgi.input':
            todelete = ('body', 'forms', 'files', 'params', 'post', 'json')
        elif key == 'QUERY_STRING':
            todelete = ('query', 'params')
        elif key.startswith('HTTP_'):
            todelete = ('headers', 'cookies')

        for key in todelete:
            self.environ.pop('bottle.request.' + key, None)
    def __repr__(self):
        return '<%s: %s %s>' % (self.__class__.__name__, self.method, self.url)

    def __getattr__(self, name):
        """ Search in self.environ for additional user defined attributes. """
        try:
            var = self.environ['bottle.request.ext.%s' % name]
            return var.__get__(self) if hasattr(var, '__get__') else var
        except KeyError:
            raise AttributeError('Attribute %r not defined.' % name)

    def __setattr__(self, name, value):
        if name == 'environ': return object.__setattr__(self, name, value)
        key = 'bottle.request.ext.%s' % name
        if key in self.environ:
            raise AttributeError("Attribute already defined: %s" % name)
        self.environ[key] = value

    def __delattr__(self, name, value):
        try:
            del self.environ['bottle.request.ext.%s' % name]
        except KeyError:
            raise AttributeError("Attribute not defined: %s" % name)
```

上面定义的方法使得 `BaseRequest` 对象可以像字典一点使用，所有的键值对实际会保存到内部属性  `self.environ` 中，除此之外 `BaseRequest` 还定义了很多便于访问的属性，如访问 `Content-Type` 首部：

```python
    @property
    def content_type(self):
        """ The Content-Type header as a lowercase-string (default: empty). """
        return self.environ.get('CONTENT_TYPE', '').lower()
```

一些不能直接从 `environ` 字典中获取的变量，bottle 使用专门的访问函数，将其以 `bottle.request.XXX` 的形式保存到 `environ` 中，便于以后访问，bottle 使用自定义的 `DictProperty` 完成此功能：

```python
    @DictProperty('environ', 'bottle.request.urlparts', read_only=True)
    def urlparts(self):
        """ The :attr:`url` string as an :class:`urlparse.SplitResult` tuple.
            The tuple contains (scheme, host, path, query_string and fragment),
            but the fragment is always empty because it is not visible to the
            server. """
        env = self.environ
        http = env.get('HTTP_X_FORWARDED_PROTO') \
             or env.get('wsgi.url_scheme', 'http')
        host = env.get('HTTP_X_FORWARDED_HOST') or env.get('HTTP_HOST')
        if not host:
            # HTTP 1.1 requires a Host-header. This is for HTTP/1.0 clients.
            host = env.get('SERVER_NAME', '127.0.0.1')
            port = env.get('SERVER_PORT')
            if port and port != ('80' if http == 'http' else '443'):
                host += ':' + port
        path = urlquote(self.fullpath)
        return UrlSplitResult(http, host, path, env.get('QUERY_STRING'), '')
```

`DictProperty()` 是一个自定义的 [descripter object](http://docs.python.org/2/reference/datamodel.html#implementing-descriptors)，bottle 使用装饰器的模式将 getter 函数挂接到 descripter 的 `__get__()` 方法中(类似于 Python 自带的 `property` 装饰器）：

```python
class DictProperty(object):
    """ Property that maps to a key in a local dict-like attribute. """

    def __init__(self, attr, key=None, read_only=False):
        self.attr, self.key, self.read_only = attr, key, read_only

    def __call__(self, func):      # 将 DictProperty 作为装饰器使用
        functools.update_wrapper(self, func, updated=[])
        self.getter, self.key = func, self.key or func.__name__
        return self

    def __get__(self, obj, cls):
        if obj is None: return self
        key, storage = self.key, getattr(obj, self.attr)
        if key not in storage: storage[key] = self.getter(obj)
        return storage[key]

    def __set__(self, obj, value):
        if self.read_only: raise AttributeError("Read-Only property.")
        getattr(obj, self.attr)[self.key] = value

    def __delete__(self, obj):
        if self.read_only: raise AttributeError("Read-Only property.")
        del getattr(obj, self.attr)[self.key]
```

使用 `@DictProperty('environ', 'bottle.request.XXX')` 时，实际上将被装饰函数的返回值保存到了 `self.environ['bottle.request.XXX']` 中。

我们再来看全局变量 `request` 的类定义：

```python
class LocalRequest(BaseRequest):
    bind = BaseRequest.__init__    # 这个方法用于对 request 对象进行初始化
    environ = _local_property()
``` 

这个类直接继承于 `BaseRequest`，新增了一个 `bind()` 方法用于初始化，同时使用 `_local_property()` 函数设置了线程的私有属性。`_local_property()` 函数内部维护线程私有的数据（这样确保 `request` 全局变量是线程安全的），它返回一个 `property` 对象，这个对象有相应的 `getter`, `setter` 和 `deleter` 方法：

```python
def _local_property():
    ls = threading.local()   # 确保线程安全

    def fget(_):
        try:
            return ls.var
        except AttributeError:
            raise RuntimeError("Request context not initialized.")

    def fset(_, value):
        ls.var = value

    def fdel(_):
        del ls.var

    return property(fget, fset, fdel, 'Thread-local property')  # 返回一个 property 对象
```

`reponse` 变量类似于 `request` 变量，它使用 `BaseResponse` 作为基类，内部封装了响应的状态码，header, cookie 等信息，这里不再赘述。


# Application

一个 Web 框架必须向 WSGI Server 提供 application 对象，用于处理用户的请求。在 Bottle 中，application 对象就是一个 `Bottle` 实例，所有的 application 保存在 `AppStack` 中。

`Bottle` 这个类中定义了很多方法，因为 application 对象必须是一个 callable object，我们可以从这个类的调用函数开始，一步步分析它是如何处理一个请求的，`Bottle` 的调用方法为：

```python
    def __call__(self, environ, start_response):
        """ Each instance of :class:'Bottle' is a WSGI application. """
        return self.wsgi(environ, start_response)
```

这里直接调用了 `wsgi` 方法：

```python
    def wsgi(self, environ, start_response):
        """ The bottle WSGI-interface. """
        try:
            out = self._cast(self._handle(environ))   # 处理请求，生成响应结果
            # rfc2616 section 4.3
            if response._status_code in (100, 101, 204, 304)\
            or environ['REQUEST_METHOD'] == 'HEAD':
                if hasattr(out, 'close'): out.close()
                out = []
            start_response(response._status_line, response.headerlist)  # 进行响应
            return out
    ...（省略）
```

`wsgi()` 方法用于生成输出，调用 `start_reponse()` 回调函数通过 WSGI Server 进行响应，请求的具体处理在 `_handle()` 方法中：

```python
    def _handle(self, environ):
        ...(略)

        environ['bottle.app'] = self
        request.bind(environ)          # 使用 environ 初始化当前请求的 request 对象
        response.bind()                # 初始化当前请求的 response 对象

        try:
            while True: # Remove in 0.14 together with RouteReset
                out = None
                try:
                    self.trigger_hook('before_request')
                    route, args = self.router.match(environ)   # 进行路由匹配
                    environ['route.handle'] = route
                    environ['bottle.route'] = route
                    environ['route.url_args'] = args
                    out = route.call(**args)     # 调用对应的处理函数生成响应结果
                    break
        ...(略)
        return out
```

`_handle()` 方法在处理请求的关键步骤有两个，首先使用 `router` 对请求进行路由匹配，得到对应的路由，生成相应的参数：

```python
route, args = self.router.match(environ)
```

然后调用路由 `route` 所关联的处理函数进行处理，得到返回结果：

```python
out = route.call(**args)
```

到这里，一个请求的处理大体流程就清晰了：

1. 启动 WSGI Server
2. 接收请求，调用给 Bottle 对象的 `__call__` 进行处理
3. bottle 根据传递的 `environ` 字典，初始化 `request` 和 `response` 对象
4. 根据 URL 匹配路由器对象
5. 调用对应路由对象关联的处理函数，返回处理结果
6. 将结果封装在 `reponse` 对象中返回给 WSGIServer

这里简略了模板处理，session 处理，异常错误处理等细节。可以看到一个处理究竟如何处理，很大程度取决于上面的 4，5 两步，这两步由 bottle 路由来完成。

# 路由

从前面的分析可以得知，`Bottle` 类内部使用了连个关键的对象来进行路由匹配，即 `router` 和 `route` 对象。来看 `Bottle` 的初始化方法：

```python
class Bottle(object):
    ...(略)
    def __init__(self, **kwargs):
        ...(略)
        self.routes = []  # List of installed :class:`Route` instances.
        self.router = Router()  # Maps requests to :class:`Route` instances.
        ...(略)
```

`self.routes` 是一个列表，保存了这个应用所有的 `Route` 对象，`self.router` 是一个 `Router` 实例，它用于将请求映射到对应的 `Route` 对象，再由 `Route` 对象调用对应的处理函数处理请求。

我们再来看看 Bottle 是如果将 URL 与处理函数进行关联的：

```python
from bottle import route

@route('/hello')
def hello():
    return "Hello World!"
```

如果在代码中显式的初始化了 application 对象：

```python
rom bottle import Bottle, run

app = Bottle()

@app.route('/hello')
def hello():
    return "Hello World!"
```

实际上，这两种方式是相同的，后者显示的调用了 `app` 对象的 `route()` 方法，前者实际上使用的就是默认 `app` 的 `route` 方法。

```python
def make_default_app_wrapper(name):
    """ Return a callable that relays calls to the current default app. """

    @functools.wraps(getattr(Bottle, name))
    def wrapper(*a, **ka):
        return getattr(app(), name)(*a, **ka)    # 使用 app() 从 AppStack 中获取默认的 app

    return wrapper

route     = make_default_app_wrapper('route')
```

因此，直接使用 `route()` 就是调用了 `app().route()`. 我们来看 `Bottle` 的 `route()` 方法：

```python
    def route(self,
              path=None,
              method='GET',
              callback=None,
              name=None,
              apply=None,
              skip=None, **config):
        if callable(path): path, callback = None, path
        plugins = makelist(apply)
        skiplist = makelist(skip)

        def decorator(callback):
            if isinstance(callback, basestring): callback = load(callback)
            for rule in makelist(path) or yieldroutes(callback):
                for verb in makelist(method):
                    verb = verb.upper()
                    route = Route(self, rule, verb, callback,
                                  name=name,
                                  plugins=plugins,
                                  skiplist=skiplist, **config)
                    self.add_route(route)
            return callback

        return decorator(callback) if callback else decorator
```

`route()` 方法是一个带参数的装饰器，使用 `@route()` 装饰器时，它的主要功能是：

```python
route = Route(self, rule, verb, callback,
              name=name,
              lugins=plugins,
              skiplist=skiplist, **config)
self.add_route(route)
``` 

通过 `add_route` 方法，代码中所有被装饰器修饰的函数在运行时都被注册到了 application 对象中，具体来说是 `Bottle.routes` 和 `Bottle.router` 中。

即根据装饰器参数中的 `path` 等参数，生成 `Route` 对象，这个 `Route` 对象包含了 URL 路径和对应的回调函数，然后调用 `add_route` 方法，将这个 `Route` 加入到 `self.routes` 列表，同时在 `self.router` 中注册此对象：

```python
    def add_route(self, route):
        """ Add a route object, but do not change the :data:`Route.app`
            attribute."""
        self.routes.append(route)
        self.router.add(route.rule, route.method, route, name=route.name)
        if DEBUG: route.prepare()
```

`Route` 对象封装了 URL 路径，对应的回调函数等信息，在此不做赘述。`add_route` 是一个比较复杂的方法，它主要完成：

* 使用正则表达式对 `route` 中注册的 URL path 进行解析
* 判断 URL 是静态还是动态，以及 URL path 的 `mode` 和 `config` 参数
* 生成正则表达式，记录正则表达式的匹配名称，这些名称会被作为参数传递给对应的回调函数
* 生成 in/out filter，用于对 URL path 中的参数进行格式化，对于普通的 URL route 规则，in filter 为 `None`，out filter 为 `str`
* 将 `route` 中的方法，路径，正则表达式，filter 等信息记录到 `router` 实例属性中
* 编译正则表达式，生成 `match` 方法和 `getargs` 方法，`getargs` 方法用于在匹配之后，将匹配的组名称结果转换为关参数，这些参数会传递给对应的回调函数

对于静态路由规则，不需要正则表达式。对于普通类型的动态路由使用，如：

```python
@app.route('/hello/<name>')
def callback(name):
    return "Hello {!r}!".format(name)
```

`Router` 会在内部使用默认的正则表达式模式，默认的匹配规则为：`'[^/]+'`。

在收到一个请求时，Bottle app 会调用 `Router.match()` 对请求的方法和 URL 进行匹配：

```python
route, args = self.router.match(environ)
```

返回的结果接收对应的 `Route` 对象和对应回调函数所需的参数。`match()` 方法是这样的：

```python
    def match(self, environ):
        """ Return a (target, url_args) tuple or raise HTTPError(400/404/405). """
        verb = environ['REQUEST_METHOD'].upper()
        path = environ['PATH_INFO'] or '/'

        if verb == 'HEAD':
            methods = ['PROXY', verb, 'GET', 'ANY']
        else:
            methods = ['PROXY', verb, 'ANY']

        for method in methods:
            if method in self.static and path in self.static[method]:
                target, getargs = self.static[method][path]
                return target, getargs(path) if getargs else {}
            elif method in self.dyna_regexes:
                for combined, rules in self.dyna_regexes[method]:
                    match = combined(path)
                    if match:
                        target, getargs = rules[match.lastindex - 1]
                        return target, getargs(path) if getargs else {}
        ...(略)
```

在进行 URL 匹配时，有两种情况：

* URL 可以直接匹配注册的静态路由
* URL 需要镜像动态匹配，调用编译好的正则表达式 `match()` 方法进行匹配

最终返回 `Route` 对象和回调函数的参数。 Bottle app 在完成匹配后，会调用 `Route` 对象的 `call()` 方法，这个方法实际被转发到了相关联的回调函数，同时会将所需的参数传递给回调函数：

```python
out = route.call(**args)
```

最后，处理的结果会经过 `Bottle._cast()` 进行处理结果转换为符合WSGI标准的结果，最终返回至客户端。

# 其他

除了上面的主要功能之外，bottle 还有一些特性：

* 支持 hooks，在请求处理前/后调用 hooks 函数
* plugins，支持插件，完成一些特殊功能
* 模板，调用内置模板或者 jinja/mako 等模板引擎

由于篇幅限制，这里没有介绍 bottle 的一些高级功能和插件功能，需要读者自行探索。阅读 bottle 的源码能够帮助了解一个 Web 框架的实现细节以及工作流程，但是需要指出的是，bottle 虽然精简，但是其代码规范性做的并不好，主要有以下问题：

* 不遵循 PEP8 规范，经常将两行代码写在一行
* 正则表达式的处理缺少必要的注释，读者只能靠官方文档自行猜测
* python2/3 的兼容性处理都是通过手动的方式完成的，增加了冗余代码，并没有使用 `six` 这种兼容库

注：此文参考的是 bottle `5a6fc77c9c57501b08e5b4f9161ae07d277effa9` 版本的代码。

