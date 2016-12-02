gunicorn处理WSGI request
==========================

WSGI的标准在PEP333中, 以下是摘要

environ Variables
~~~~~~~~~~~~~~~~~~

The environ dictionary is required to contain these CGI environment variables, as defined by the Common Gateway Interface specification [2] . The following variables must be present, unless their value would be an empty string, in which case they may be omitted, except as otherwise noted below.

REQUEST_METHOD
+++++++++++++++++

The HTTP request method, such as "GET" or "POST" . This cannot ever be an empty string, and so is always required.

SCRIPT_NAME
++++++++++++++

The initial portion of the request URL's "path" that corresponds to the application object, so that the application knows its virtual "location". This may be an empty string, if the application corresponds to the "root" of the server.

PATH_INFO
+++++++++++++

The remainder of the request URL's "path", designating the virtual "location" of the request's target within the application. This may be an empty string, if the request URL targets the application root and does not have a trailing slash.

QUERY_STRING
+++++++++++++++

The portion of the request URL that follows the "?" , if any. May be empty or absent.

CONTENT_TYPE
++++++++++++++++

The contents of any Content-Type fields in the HTTP request. May be empty or absent.

CONTENT_LENGTH
+++++++++++++++

The contents of any Content-Length fields in the HTTP request. May be empty or absent.

SERVER_NAME , SERVER_PORT
++++++++++++++++++++++++++++++

When combined with SCRIPT_NAME and PATH_INFO , these variables can be used to complete the URL. Note, however, that HTTP_HOST , if present, should be used in preference to SERVER_NAME for reconstructing the request URL. See the URL Reconstruction section below for more detail. SERVER_NAME and SERVER_PORT can never be empty strings, and so are always required.

SERVER_PROTOCOL
+++++++++++++++++++

The version of the protocol the client used to send the request. Typically this will be something like "HTTP/1.0" or "HTTP/1.1" and may be used by the application to determine how to treat any HTTP request headers. (This variable should probably be called REQUEST_PROTOCOL , since it denotes the protocol used in the request, and is not necessarily the protocol that will be used in the server's response. However, for compatibility with CGI we have to keep the existing name.)

HTTP_Variables
++++++++++++++++++

Variables corresponding to the client-supplied HTTP request headers (i.e., variables whose names begin with "HTTP_" ). The presence or absence of these variables should correspond with the presence or absence of the appropriate HTTP header in the request.



1. gunicorn.sync.SyncWorker.handle
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class SyncWorker(base.Worker):

        def handle(self, listener, client, addr):
            req = None
            try:
                if self.cfg.is_ssl:
                    client = ssl.wrap_socket(client, server_side=True,
                        **self.cfg.ssl_options)

                # 解析request
                parser = http.RequestParser(self.cfg, client)
                # 这里的next会掉parser的__next__方法
                req = six.next(parser)
                self.handle_request(listener, req, client, addr)
            excpet:
                pass


2. gunicorn.http.parser.Parser
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class Parser(object):
        
        def next(self):
            # self.mesg_class = gunicorn.http.message.Request
            self.mesg = self.mesg_class(self.cfg, self.unreader, self.req_count)
            if not self.mesg:
                raise StopIteration()
            return self.mesg

3. gunicorn.http.message.Request
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class Message(object):

        def parse_headers(self, data):
            headers = []
            # Split lines on \r\n keeping the \r\n on each line
            lines = [bytes_to_str(line) + "\r\n" for line in data.split(b"\r\n")]

            # Parse headers into key/value pairs paying attention
            # to continuation lines.
            while len(lines):
                # 解析每一行
                curr = lines.pop(0)
                header_length = len(curr)
                if curr.find(":") < 0:
                    raise InvalidHeader(curr.strip())
                name, value = curr.split(":", 1)
                # 把头部的名称都大写
                name = name.rstrip(" \t").upper()
            # headers = [{'HOST': 'xxx'}, {'X-FORWARD': 'xxx'}, ...]
            return headers
            

    class Request(Message):
        def __init__(self, cfg, unreader, req_number=1):
            # 这里的调用Message的__init__方法, 然后会调到自己的parse方法
            super(Request, self).__init__(cfg, unreader)
    
        # 解析http头部
        def parse(self, unreader):
            buf = BytesIO()
            self.get_data(unreader, buf, stop=True)
    
            # get request line
            # 读取第一行, METHOD PATH HTTP_VERSION
            line, rbuf = self.read_line(unreader, buf, self.limit_request_line)
    
            # proxy protocol
            if self.proxy_protocol(bytes_to_str(line)):
                # get next request line
                buf = BytesIO()
                buf.write(rbuf)
                line, rbuf = self.read_line(unreader, buf, self.limit_request_line)
    
            # 解析第一行
            self.parse_request_line(bytes_to_str(line))
            buf = BytesIO()
            buf.write(rbuf)
    
            data = buf.getvalue()
            idx = data.find(b"\r\n\r\n")
    
            done = data[:2] == b"\r\n"
            while True:
                idx = data.find(b"\r\n\r\n")
                done = data[:2] == b"\r\n"
    
                if idx < 0 and not done:
                    self.get_data(unreader, buf)
                    data = buf.getvalue()
                    if len(data) > self.max_buffer_headers:
                        raise LimitRequestHeaders("max buffer headers")
                else:
                    break
    
            if done:
                self.unreader.unread(data[2:])
                return b""
    
            # 解析头部, self.parse_headers = gunicorn.http.message.Message.parse_headers
            self.headers = self.parse_headers(data[:idx])
    
            ret = data[idx + 4:]
            buf = BytesIO()
            return ret
        


4. gunicorn.sync.SyncWorker.handle_request
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class SyncWorker(base.Worker):

        def handle_request(self, listener, req, client, addr):
            environ = {}
            resp = None
            try:
                # self.cfg.pre_request
                self.cfg.pre_request(self, req)
                request_start = datetime.now()
                # req.headers就是上面解析出来的头部
                # 这里会把environ传递给django的request/drf的request
                # 并且这里的wsgi.create会把HTTP_前缀加到http头的key上, 除了一些指定的头部之外
                # wsgi = gunicorn.http.wsgi.create
                resp, environ = wsgi.create(req, client, addr,
                        listener.getsockname(), self.cfg)


5 gunicorn.http.wsgi.create

除了一些指定的头部, 为其他http头部加上HTTP_前缀, 并且加上一些软件信息, 比如wsgi.url_scheme等等

.. code-block:: python

    def create(req, sock, client, server, cfg):
        resp = Response(req, sock, cfg)
    
        # set initial environ
        environ = default_environ(req, sock, cfg)
        # 解析头部
        for hdr_name, hdr_value in req.headers:
            # 遇到指定的头部, 则不加HTTP_前缀
            if hdr_name == "EXPECT":
                # handle expect
                if hdr_value.lower() == "100-continue":
                    sock.send(b"HTTP/1.1 100 Continue\r\n\r\n")
            elif secure_headers and (hdr_name in secure_headers and
                  hdr_value == secure_headers[hdr_name]):
                url_scheme = "https"
            elif hdr_name == 'HOST':
                host = hdr_value
            elif hdr_name == "SCRIPT_NAME":
                script_name = hdr_value
            elif hdr_name == "CONTENT-TYPE":
                environ['CONTENT_TYPE'] = hdr_value
                continue
            elif hdr_name == "CONTENT-LENGTH":
                environ['CONTENT_LENGTH'] = hdr_value
                continue
    
            # 除了一些制定的头部, 其他的头的key都加上HTTP_前缀
            key = 'HTTP_' + hdr_name.replace('-', '_')
            if key in environ:
                hdr_value = "%s,%s" % (environ[key], hdr_value)
            environ[key] = hdr_value
    
        # set the url schejeme
        environ['wsgi.url_scheme'] = url_scheme
    
        # set the REMOTE_* keys in environ
        # authors should be aware that REMOTE_HOST and REMOTE_ADDR
        # may not qualify the remote addr:
        # http://www.ietf.org/rfc/rfc3875
        if isinstance(client, string_types):
            environ['REMOTE_ADDR'] = client
        elif isinstance(client, binary_type):
            environ['REMOTE_ADDR'] = str(client)
        else:
            environ['REMOTE_ADDR'] = client[0]
            environ['REMOTE_PORT'] = str(client[1])
    
        # handle the SERVER_*
        # Normally only the application should use the Host header but since the
        # WSGI spec doesn't support unix sockets, we are using it to create
        # viable SERVER_* if possible.
        if isinstance(server, string_types):
            server = server.split(":")
            if len(server) == 1:
                # unix socket
                if host and host is not None:
                    server = host.split(':')
                    if len(server) == 1:
                        if url_scheme == "http":
                            server.append(80),
                        elif url_scheme == "https":
                            server.append(443)
                        else:
                            server.append('')
                else:
                    # no host header given which means that we are not behind a
                    # proxy, so append an empty port.
                    server.append('')
        environ['SERVER_NAME'] = server[0]
        environ['SERVER_PORT'] = str(server[1])
    
        # set the path and script name
        path_info = req.path
        if script_name:
            path_info = path_info.split(script_name, 1)[1]
        environ['PATH_INFO'] = unquote_to_wsgi_str(path_info)
        environ['SCRIPT_NAME'] = script_name
    
        # override the environ with the correct remote and server address if
        # we are behind a proxy using the proxy protocol.
        environ.update(proxy_environ(req))
        return resp, environ


6. django接收gunicorn的environ, 生成自己的wsgi request
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    
django.core.handlers.wsgi.WSGIHandler

.. code-block:: python

    class WSGIHandler(base.BaseHandler):
    
        # 这里的enviro就是gunicorn的生成的environ
        def __call__(self, environ, start_response):
            try:
                # 生成wsgi request
                # self.request_class = django.core.handlers.wsgi.WSGIRequest
                request = self.request_class(environ)
            except UnicodeDecodeError:
                logger.warning('Bad Request (UnicodeDecodeError)',
                    exc_info=sys.exc_info(),
                    extra={
                        'status_code': 400,
                    }
                )
                response = http.HttpResponseBadRequest()
            else:
                # 调用到dispatch, 把WSGIRequest传入给drf
                response = self.get_response(request)    


7. django.core.handlers.wsgi.WSGIRequest中, 把environ放入到自己的META中
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class WSGIRequest(http.HttpRequest):
        def __init__(self, environ):
            self.META = environ



8. django rest framework接收gunicorn生成的environ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    class Request(object):
    
        def __init__(self, request, parsers=None, authenticators=None,
                     negotiator=None, parser_context=None):
            # 这里传入的request就是WSGIRequest
            self._request = request

