## gunicorn是如何与Flask通信的

[搬运地址](https://zhuanlan.zhihu.com/p/24650254)

### 问题

Flask是在对werkzeug简单封装的基础上形成的。作为一个轻量级的web框架，Flask容易上手，学习和进阶曲线不陡峭，是Python Web框架入门的首选。
Flask自带了一个HTTP Server用于开发和调试， 如:

```python
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return 'Hello'
    
if __name__ == '__main__':
    app.run()
```

但是，受制于安全、性能等问题，该Server不能用于生产环境。
生产环境中部署Flask等WSGI常用的组合是：

```text
nginx + gunicorn + application(Flask应用等)
```

gunicorn是一款广受欢迎的WSGI服务器，可以用于部署Flask应用。
那么， gunicorn收到请求之后，使如何对请求进行解析并发给Flask应用的呢?

### 代码跟踪

“柿子先捡软的捏”，Flask代码比较简单，就先看Flask的代码。

既然是探究”gunicorn是如何和Flask应用进行通信”， 那么我们就来看一下Flask中用于创建应用的Flask类。

```python
class Flask(_PackageBoundObject):
    ...
    def dispatch_request(self):
        """Does the request dispatching. 
        """
        req = _request_ctx_stack.top.request
        if req.routing_exception is not None:
            self.raise_routing_exception(req)
        rule = req.url_rule
        # if we provide automatic options for this URL and the
        # request came with the OPTIONS method, reply automatically
        if getattr(rule, 'provide_automatic_options', False) \
           and req.method == 'OPTIONS':
            return self.make_default_options_response()
        # otherwise dispatch to the handler for that endpoint
        # view_args是动态路由参数
        return self.view_functions[rule.endpoint](**req.view_args)
    def full_dispatch_request(self):
        """Dispatches the request and on top of that performs request
        pre and postprocessing as well as HTTP exception catching and
        error handling.
        .. versionadded:: 0.7
        """
        self.try_trigger_before_first_request_functions()
        try:
            request_started.send(self)
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
        except Exception as e:
            rv = self.handle_user_exception(e)
        response = self.make_response(rv)
        response = self.process_response(response)
        request_finished.send(self, response=response)
        return response
    def wsgi_app(self, environ, start_response):
        """The actual WSGI application.  This is not implemented in
        `__call__` so that middlewares can be applied without losing a
        reference to the class.  So instead of doing this::
            app = MyMiddleware(app)
        It's a better idea to do this instead::
            app.wsgi_app = MyMiddleware(app.wsgi_app)
        Then you still have the original application object around and
        can continue to call methods on it.
        :param environ: a WSGI environment
        :param start_response: a callable accepting a status code,
                               a list of headers and an optional
                               exception context to start the response
        """
        ctx = self.request_context(environ)
        ctx.push()
        error = None
        try:
            try:
                response = self.full_dispatch_request()
            except Exception as e:
                error = e
                response = self.make_response(self.handle_exception(e))
            return response(environ, start_response)
        finally:
            if self.should_ignore_error(error):
                error = None
            ctx.auto_pop(error)
    def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`."""
        # gunicorn会调用该方法
        return self.wsgi_app(environ, start_response)
```



Flask的代码有1000+行，为了简单起见，我们只关注:

```python
wsgi_app(self, environ, start_response)
    ...
full_dispatch_request(self)
    ...
dispatch_request(self)
    ...
__call__(self, environ, start_response)
    ...
```

这四个方法，这四个方法的用途分别是什么呢?

1. **wsgi_app(…)**

> Flask支持wsgi协议，该方法是gunicorn等中间件和Flask通信的间接入口(直接入口是`__call__()`)

2. **full_dispatch_request(…)**

> 用于分发请求，包括在请求被处理前执行的操作、处理请求、返回响应、处理响应等

3. **dispatch_request(…)**

> 分发请求并处理请求

4） **`__call__(…)`**

> 是gunicorn等中间件和flask通信的直接入口, 比如声明了: app = Flask(__name__), 那么gunicorn会通过"app(...)"的方式来把请求的参数信息发给flask应用。

### gunicorn和Flask通信的代码

gunicorn是”主-从”模式，工作者进程用于接收用户请求并通知Flask应用处理，所以是工作者进程和Flask通信的。
gunicorn支持多种工作者进程，这里以Sync工作者为例，介绍一下和Flask的通信过程。

假设以：

```shell
# debug_flask是声明app = Flask(__name__)的.py文件
gunicorn  debug_flask:app
```

启动Flask应用，那么gunicorn会：

1. 声明一个WSGIApplication对象，
   1) 该对象包含一个init()方法，在该方法中，有一句代码很关键: `self.app_uri = args[0]`
   此时， `self.app_uri`的值是debug_flask:app, 因为我们是通过`gunicorn debug_flask:app`来启动应用的。也就是说，`self.app_uri`保存的是Flask应用对象的位置。
   2) 该对象包含一个load_wsgiapp(self)方法，该方法通过util.import_app(self.app_uri)来加载Flask应用对象。
   3) WSGIApplication对象**调用**run()方法来启动服务器。由于WSGIApplication是Application类的子类，而run()是在Application中定义的，所以执行的是Application的run()方法。
   
2. Application是BaseApplication的子类，在Application的run()方法会去调用BaseApplication的run()方法：

   1) BaseApplication中定义了一个wsgi(self)方法， 定义如下：

```python
def wsgi(self):
    if self.callable is None:
        self.callable = self.load()
    return self.callable
```
   也就是说，它返回Flask应用对象(很重要)。

   2) 在BaseApplication的run()中，会通过Arbiter(self).run()启动gunicorn的主进程。也就是说，主进程对象包含了一个BaseApplication对象，而BaseApplication对象可以通过wsgi()方法来获得原始的Flask应用对象。

3. 主进程对象的run()方法是一个while循环，监听时间并管理工作者进程的数量。在创建工作者进程时，

   ```text
   worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS, elf.app, self.timeout / 2.0, self.cfg, self.log)
   ```

   会把BaseApplication对象传递给工作者进程，并通过：

   ```text
   worker.init_process()
   ```

   初始化工作进程并启动工作者进程。

4. 在Worker的init_process()方法中，通过：

   ```text
   self.wsgi = self.app.wsgi()
   ```

   工作者进程就获得了原始的Flask应用对象，并保存为self.wsgi。下面以Sync工作者进程为例。

5. Sync工作者监听socket， 使用handle_request方法来获取请求，并通过：
   respiter = self.wsgi(environ, resp.start_response)
   来调用Flask的`__call__`方法来让Flask应用来处理请求，这就实现了**gunicorn和Flask应用的通信过程**。

