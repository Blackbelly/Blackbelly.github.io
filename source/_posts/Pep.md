## PEP 3333 python wsgi

此外, 为了让已有的或者未来的框架和服务能够轻易实现, wsgi 应该能够很容易的创建 request 的预处理器, response 的 post 处理器, 以及其他基于 wsgi  中间件的部件, 从而使其对于 server 来说像一个 app, 而对于 app 来说又像一个 server



### specification overview

wsgi 接口有两个方面, 服务端(网关端), 以及应用端(框架端). 服务端包含了一个能够调用的对象, 提供给 app 使用, 具体提供的对象取决于 server 端, 假设有些 server 需要 app 的部署者写一些简单的脚本去创建 server 对象, 提供 app 对象. 其他的 server 可以使用具体的配置文件或其他途径去指定 application 应该从哪导入, 或是如何获得.



此外, 纯 server 和 application 也可以创建中间件去实现这两种特性. 某些组件, 对于其内部的 server 来说, 其行为就像一个 application, 而对于其内部的 application 来说, 又像是一个 server, 并且能够提供拓展的 api 接口, 文本转换, 导航, 或者其他有用的功能



### The application / framwork side

application 对象是一个接受两个参数的简单可调用对象. 术语'对象'不应误解为一个对象实例: 函数, 方法, 类, 或者带有 `__call__` 方法的实例都可以认为是一个 application 对象. 



```
HELLO_WORLD = b"Hello world!\n"

def simple_app(environ, start_response):
    """Simplest possible application object"""
    """application 对象, 接受两个参数, 能够被调用"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return [HELLO_WORLD]

class AppClass:
    """Produce the same output, but using a class

    (Note: 'AppClass' is the "application" here, so calling it
    returns an instance of 'AppClass', which is then the iterable
    return value of the "application callable" as required by
    the spec.

    If we wanted to use *instances* of 'AppClass' as application
    objects instead, we would have to implement a '__call__'
    method, which would be invoked to execute the application,
    and we would need to create an instance for use by the
    server or gateway.
    """

    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response

    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield HELLO_WORLD
```





### The Server / Gateway Side

server or gateway 为 http client 的每个请求调起一次 application. 下面有个简单的 CGI gateway, 实现了一个带有 application 对象的函数. 需要注意的是这个简单例子的 error handle 十分有限, 默认会将未捕获到的异常抛出为 `sys.stderr` , 并通过 web server 记录日志

```
import os, sys

enc, esc = sys.getfilesystemencoding(), 'surrogateescape'

def unicode_to_wsgi(u):
    # Convert an environment variable to a WSGI "bytes-as-unicode" string
    return u.encode(enc, esc).decode('iso-8859-1')

def wsgi_to_bytes(s):
    return s.encode('iso-8859-1')

def run_with_cgi(application):
    environ = {k: unicode_to_wsgi(v) for k,v in os.environ.items()}
    environ['wsgi.input']        = sys.stdin.buffer
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
        out = sys.stdout.buffer

        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             out.write(wsgi_to_bytes('Status: %s\r\n' % status))
             for header in response_headers:
                 out.write(wsgi_to_bytes('%s: %s\r\n' % header))
             out.write(wsgi_to_bytes('\r\n'))

        out.write(data)
        out.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]

        # Note: error checking on the headers should happen here,
        # *after* the headers are set.  That way, if an error
        # occurs, start_response can only be re-called with
        # exc_info set.

        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```



