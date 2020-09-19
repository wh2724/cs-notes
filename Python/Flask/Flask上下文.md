## Flask上下文

[搬运地址](https://www.cnblogs.com/ybjourney/p/13417529.html)

### 什么是Flask中的上下文

在Flask中，对一个请求进行处理时，视图函数一般都会需要请求参数、配置等对象，当然不能对每个请求都传参一层层到视图函数（这显然很不优雅嘛），为此，设计出了上下文机制（比如像我们经常会调用的request就是上下文变量）。

Flask中提供了两种上下文：

1. **请求上下文(Request Context)**：包括request和session，保存请求相关的信息

2. **应用上下文(Application Context)**：包括current_app和g，为了更好的分离程序的状态，应用起来更加灵活，方便调测

- application指的是`app=Flask(name)`创建的这个对象app
- request指的是每次http请求发生时，WSGI server(比如gunicorn)调用`Flask.__call__()`之后，在Flask对象内部创建的Request对象
- application表示用于响应WSGI请求的应用本身，request 表示每次http请求

**这四个是上下文变量具体的作用是什么？**

1. **request**：封装客户端发送的请求报文数据
2. **session**：用于记住请求之间的数据，通过签名的cookie实现，常用来记住用户登录状态
3. **current_app**：指向处理请求的当前程序实例，比如获取配置，经常会用current_app.config
4. **g**：当前请求中的全局变量，因为程序上下文的生命周期是伴随请求上下文产生和销毁的，所以每次请求都会重设；一般结合钩子函数在请求处理前使用。

### 具体实现

上下文具体的实现文件：[ctx.py](https://github.com/pallets/flask/blob/master/src/flask/ctx.py)

请求上下文对象通过RequestContext类实现，当Flask程序收到请求时，会在wsgi_app()中调用Flask.request_context()，实例化RequestContext()作为请求上下文对象，接着会通过push()方法将请求数据推入到请求上下文堆栈(LocalStack)，然后通过full_dispatch_request执行视图函数，调用完成之后通过auto_pop方法来移除。所以，请求上下文的生命周期开始于调用wsgi_app()时，结束与响应生成之后。具体代码：

```python
def wsgi_app(self, environ, start_response):
    
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:  # noqa: B001
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

应用上下文对象通过AppContext类实现，应用上下文的创建方式有两种：

1. 自动创建：在处理请求时，程序上下文会随着请求上下文一起被创建
2. 手动创建：with语句

通过阅读源码，可以看到上面两个上下文对象的push和pop都是通过操作LocalStack对象实现的，那么，**LocalStack是怎样实现的呢？**

Werkzeug的LocalStack是栈结构，在 globals.py中定义：

```
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
```

具体的实现：

```python
class LocalStack(object):

    def __init__(self):
        self._local = Local()

    def __release_local__(self):
        self._local.__release_local__()

    def _get__ident_func__(self):
        return self._local.__ident_func__

    def _set__ident_func__(self, value):
        object.__setattr__(self._local, '__ident_func__', value)
    __ident_func__ = property(_get__ident_func__, _set__ident_func__)
    del _get__ident_func__, _set__ident_func__

    def __call__(self):
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError('object unbound')
            return rv
        return LocalProxy(_lookup)

    def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        """The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```

可以看到：

1. LocalStack实现了栈的push、pop和获取栈顶数据的top数据
2. 整个类基于Local类，在构造函数中创建Local类的实例_local，数据是push到Werkzeug提供的Local类中
3. 定义`__call__`方法，当实例被调用直接返回栈顶对象的Werkzeug提供的LocalProxy代理，即LocalProxy实例，所以，`_request_ctx_stack`和`_app_ctx_stack`都是代理。

看到这里，就有以下问题：

**Local类是怎样存储数据的呢？为啥需要存储到Local中？**

先看下代码：

```python
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident


class Local(object):
    __slots__ = ("__storage__", "__ident_func__")

    def __init__(self):
        object.__setattr__(self, "__storage__", {})
        object.__setattr__(self, "__ident_func__", get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

可以看到，Local构造函数中定义了两个属性：

1. `__storage__`：用来保存每个线程的真实数据，对应的存储结构为->`{线程ID:{name:value}}`
2. `__ident_func__`：通过get_ident()方法获取线程ID，可以看到优先会使用Greenlet获取协程ID，其次是thread模块的线程ID

Local类在保存数据的同时，记录对应的线程ID，获取数据时根据当前线程的id即可获取到对应数据，这样就保证了全局使用的上下文对象不会在多个线程中产生混乱，保证了每个线程中上下文对象的独立和准确。

可以看到，Local类实例被调用时也同样的被包装成了一个LocalProxy代理，**为什么要用LocalProxy代理？**

代理是一种设计模式，通过创建一个代理对象来操作实际对象，简单理解就是使用一个中间人来转发操作，Flask上下文处理为什么需要它？

看下代码实现：

```Python
@implements_bool
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__', '__wrapped__')

    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
        if callable(local) and not hasattr(local, '__release_local__'):
            object.__setattr__(self, '__wrapped__', local)

    def _get_current_object(self):
        """
        获取被代理的实际对象
        """
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)
            
	@property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __repr__(self):
        try:
            obj = self._get_current_object()
        except RuntimeError:
            return '<%s unbound>' % self.__class__.__name__
        return repr(obj)

    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False

    def __unicode__(self):
        try:
            return unicode(self._get_current_object())  # noqa
        except RuntimeError:
            return repr(self)

    def __dir__(self):
        try:
            return dir(self._get_current_object())
        except RuntimeError:
            return []

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]
    ...
```

通过`__getattr__()`、`__setitem__()`和`__delitem__()`会动态的更新实例对象。

再结合上下文对象的调用：

```Python
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, "request"))
session = LocalProxy(partial(_lookup_req_object, "session"))
g = LocalProxy(partial(_lookup_app_object, "g"))
```

我们可以很明确的看到：因为上下文的推送和删除是动态进行的，所以使用代理来动态的获取上下文对象。