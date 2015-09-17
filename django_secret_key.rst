Django Secret key
===================

**Django 1.7版本**

django secret key 用于django中所有的签名加密, 包括:

1. All sessions if you are using any other session backend than django.contrib.sessions.backends.cache, or if you use SessionAuthenticationMiddleware and are using the default
   get_session_auth_hash().

   默认的session认证, 除了sessoin的backend选的cache之外

2. All messages if you are using CookieStorage or FallbackStorage.

   Message框架中, 用Cookiestorage和Fallbackstorage作为存储的message

3. Form wizard progress when using cookie storage with django.contrib.formtools.wizard.views.CookieWizardView.

   表单向导中使用django.contrib.formtools.wizard.views.CookieWizardView作为cookie存储工具.

4. All password_reset() tokens.

   重设密码中所有使用到的令牌.

5. in progress form previews.

   form previews中

6. Any usage of cryptographic signing, unless a different key is provided.

   所有的加密签名

If you rotate your secret key, all of the above will be invalidated. Secret keys are not used for passwords of users and key rotation will not affect them.

用户密码并不是用secret key来加密签名的, 所以用户密码并不受影响.

django middleware
-------------------

django middleware 的文档在 `这里 <https://docs.djangoproject.com/en/1.7/topics/http/middleware/>`_

django在django.core.handlers.base.py中定义了自己的wsgi handler, 在get_response方法中处理了一个完整的request请求.

django settings中的middleware中是顺序调用, 只有处理response对象的时候, 是倒序调用middleware的process_response方法, 并且, 一旦一个middleware中返回了一个response对象,
则会倒序调用所有的middleware中的process_response方法.

1. 处理request, 首先会调用settings中middleware的process_request方法, 若返回None, 则继续调用下一个middleware的process_request, 若返回一个response对象, 则直接调用middleware中的
   process_response方法, 返回response对象.

2. 调用指定的view之前, 调用middleware中的process_view, CsrfViewMiddleware就是在这里对csrf进行检查. 同样, 返回None或者一个response对象.

3. 调用指定的view, 若出现exception, 则调用middleware中的process_exception方法, process_exception返回None或者response对象.

4. 若view返回的response有render方法, 则调用middleware中的process_template_response方法.

5. 最后, 反向调用middleware中的process_response方法, 返回response对象.

Session的认证过程
------------------

sessoin的认证过程是在SessionMiddleware, AuthenticationMiddleware以及SessionAuthenticationMiddleware中完成的(不过SessionAuthenticationMiddleware中的process_request方法只是pass掉了).

SessionMiddleware根据request中的sessionid, 初始化一个存储session的对象.

.. code-block:: python

    class SessionMiddleware(object):
        def __init__(self):
            engine = import_module(settings.SESSION_ENGINE)
            self.SessionStore = engine.SessionStore

        def process_request(self, request):
            session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME, None)
            request.session = self.SessionStore(session_key)

AuthenticationMiddleware中则根据request中的session来获取用户信息

.. code-block:: python

    from django.contrib import auth

    class AuthenticationMiddleware(object):
        def process_request(self, request):
            assert hasattr(request, 'session'), (
                "The Django authentication middleware requires session middleware "
                "to be installed. Edit your MIDDLEWARE_CLASSES setting to insert "
                "'django.contrib.sessions.middleware.SessionMiddleware' before "
                "'django.contrib.auth.middleware.AuthenticationMiddleware'."
            )
            request.user = SimpleLazyObject(lambda: get_user(request))

get_user方法中, 直接去request.session中获取用户信息

.. code-block:: python

    def get_user(request):
        """
        Returns the user model instance associated with the given request session.
        If no user is retrieved an instance of `AnonymousUser` is returned.
        """
        from .models import AnonymousUser
        user = None
        try:
            user_id = request.session[SESSION_KEY]
            backend_path = request.session[BACKEND_SESSION_KEY]
        except KeyError:
            pass
        else:
            if backend_path in settings.AUTHENTICATION_BACKENDS:
                backend = load_backend(backend_path)
                user = backend.get_user(user_id)
                # Verify the session
                if ('django.contrib.auth.middleware.SessionAuthenticationMiddleware'
                        in settings.MIDDLEWARE_CLASSES and hasattr(user, 'get_session_auth_hash')):
                    session_hash = request.session.get(HASH_SESSION_KEY)
                    session_hash_verified = session_hash and constant_time_compare(
                        session_hash,
                        user.get_session_auth_hash()
                    )
                    if not session_hash_verified:
                        request.session.flush()
                        user = None

        return user or AnonymousUser()

认证的过程在django.contrib.session.backend.base.SessionBase中.

若从request获取的sessionid为None, 则是匿名用户, get_user中直接返回匿名用户.

若从request获取的session不为None, 则会对调用self.load()根据sessionid来获取session_data, 在django.contrib.session.backend.db为例, 

.. code-block:: python

    def load(self):
        try:
            s = Session.objects.get(
                session_key=self.session_key,
                expire_date__gt=timezone.now()
            )
            return self.decode(s.session_data)
        except (Session.DoesNotExist, SuspiciousOperation) as e:
            if isinstance(e, SuspiciousOperation):
                logger = logging.getLogger('django.security.%s' %
                        e.__class__.__name__)
                logger.warning(force_text(e))
            self.create()
            return {}

这里会根据sessionid去搜索session_data, 之后解码, 并且比较hash值, 解码的过程是

.. code-block:: python

    def _hash(self, value):
        key_salt = "django.contrib.sessions" + self.__class__.__name__
        return salted_hmac(key_salt, value).hexdigest()

    def decode(self, session_data):
        encoded_data = base64.b64decode(force_bytes(session_data))
        try:
            # could produce ValueError if there is no ':'
            hash, serialized = encoded_data.split(b':', 1)
            expected_hash = self._hash(serialized)
            if not constant_time_compare(hash.decode(), expected_hash):
                raise SuspiciousSession("Session data corrupted")
            else:
                return self.serializer().loads(serialized)
        except Exception as e:
            # ValueError, SuspiciousOperation, unpickling exceptions. If any of
            # these happen, just return an empty dictionary (an empty session).
            if isinstance(e, SuspiciousOperation):
                logger = logging.getLogger('django.security.%s' %
                        e.__class__.__name__)
                logger.warning(force_text(e))
            return {}

解码是用base64解码, 解码的结构是一个hash值和一个用户字典, 这个用户字典就是存储了用户信息, 例如

c18d7fcf2a1b493fe2ab8c057e80d98bb8716dfc:{"_auth_user_hash":"eda2f06ebfbafd4b0cfb743405d64413835717bf","_auth_user_backend":"django.contrib.auth.backends.ModelBackend","_auth_user_id":1}

比较的过程是比较用户字典中的hash值和hash之后的用户字典的hash值. hash的过程是一个经过salt之后的hmac(hash message authentication code)值.

salted_hmac方法在django.util.crypto中, 使用到了settings.SECRET_KEY作为sha1的key中的一个值. 

.. code-block:: python

    def salted_hmac(key_salt, value, secret=None):
        """
        Returns the HMAC-SHA1 of 'value', using a key generated from key_salt and a
        secret (which defaults to settings.SECRET_KEY).

        A different key_salt should be passed in for every application of HMAC.
        """
        if secret is None:
            secret = settings.SECRET_KEY

        key_salt = force_bytes(key_salt)
        secret = force_bytes(secret)

        # We need to generate a derived key from our base key.  We can do this by
        # passing the key_salt and our base key through a pseudo-random function and
        # SHA1 works nicely.
        key = hashlib.sha1(key_salt + secret).digest()

        # If len(key_salt + secret) > sha_constructor().block_size, the above
        # line is redundant and could be replaced by key = key_salt + secret, since
        # the hmac module does the same thing for keys longer than the block size.
        # However, we need to ensure that we *always* do this.
        return hmac.new(key, msg=force_bytes(value), digestmod=hashlib.sha1)

而在django.contrib.session.backends.cache中的SessionStore的load方法并没有decode的过程, 也就没有hash的过程, 所以settings.SECRET_KEY并不会影响到cache backend的使用.

django登陆
~~~~~~~~~~~

登陆的过程类似, 登陆函数是在django.contrib.auth.views中的login, bakcend以django.contrib.session.backend.db为例

.. code-block:: python

    def login(request, user):
        """
        Persist a user id and a backend in the request. This way a user doesn't
        have to reauthenticate on every request. Note that data set during
        the anonymous session is retained when the user logs in.
        """
        session_auth_hash = ''
        if user is None:
            user = request.user
        if hasattr(user, 'get_session_auth_hash'):
            session_auth_hash = user.get_session_auth_hash()

        if SESSION_KEY in request.session:
            if request.session[SESSION_KEY] != user.pk or (
                    session_auth_hash and
                    request.session.get(HASH_SESSION_KEY) != session_auth_hash):
                # To avoid reusing another user's session, create a new, empty
                # session if the existing session corresponds to a different
                # authenticated user.
                request.session.flush()
        else:
            request.session.cycle_key()
        request.session[SESSION_KEY] = user.pk
        request.session[BACKEND_SESSION_KEY] = user.backend
        request.session[HASH_SESSION_KEY] = session_auth_hash
        if hasattr(request, 'user'):
            request.user = user
        rotate_token(request)
        user_logged_in.send(sender=user.__class__, request=request, user=user)

其中, user是loginform中传过来的user对象, 首先获取user的一个auth_hash, user.get_session_auth_hash中, 也是一个salted_hmac值, 只不过是用user的密码来作为一个key的salted_hmac值

.. code-block:: python

    def get_session_auth_hash(self):
        """
        Returns an HMAC of the password field.
        """
        key_salt = "django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash"
        return salted_hmac(key_salt, self.password).hexdigest()

第一次登陆(包括过期之后第一次登陆)成功后, request.session生成了一个sessionid, 然后设置用户字典, 然后调用request.session.save来保存session数据.

save方法中, 会调用django.session.backend.base.SessionBase中的encode方法来编码session_data为一个hash:{'_auth_user_id: ...', ...}形式的数据, 其中hash是后面用户字典的hash值, 用于校验

.. code-block:: python

    def encode(self, session_dict):
        "Returns the given session dictionary serialized and encoded as a string."
        serialized = self.serializer().dumps(session_dict)
        hash = self._hash(serialized)
        return base64.b64encode(hash.encode() + b":" + serialized).decode('ascii')


Message框架
--------------

django的 `messaage框架文档 <https://docs.djangoproject.com/en/1.7/ref/contrib/messages/>`_

message框架主要是将后台产生的提示性信息暂时性存储在request._message中, 以便客户端可以获取这些提示性信息的功能.

django中提供了对message操作的api, 例如:

.. code-block:: python

    from django.contrib import messages
    from django.contrib.messages import get_messages

    messages.add_message(request, level, message, extra_tags=extra_tags,
            fail_silently=fail_silently)

    storage = get_messages(request)
    for message in storage:
        do_something_with_the_message(message)

而在get_message中, 直接去获取request._message

.. code-block:: python

    def get_messages(request):
        """
        Returns the message storage on the request if it exists, otherwise returns
        an empty list.
        """
        if hasattr(request, '_messages'):
            return request._messages
        else:
            return []

message的使用需要django.contrib.messages.middleware.MessageMiddleware这个message中间件的, 这个中间件的process_request的时候, 初始化一个message的存储对象, 赋给request._message, 
之后message都存储到该对象中.在process_response的时候, 把所有的message都update到指定的地方, 例如django.contrib.messages.storage.cookie.CookieStorage将message都update到response的cookie中,
django.contrib.messages.storage.session.SessionStorage则是把message都update到request.session中.

.. code-block:: python

    class MessageMiddleware(object):
        """
        Middleware that handles temporary messages.
        """

        def process_request(self, request):
            request._messages = default_storage(request)

        def process_response(self, request, response):
            """
            Updates the storage backend (i.e., saves the messages).

            If not all messages could not be stored and ``DEBUG`` is ``True``, a
            ``ValueError`` is raised.
            """
            # A higher middleware layer may return a request which does not contain
            # messages storage, so make no assumption that it will be there.
            if hasattr(request, '_messages'):
                unstored_messages = request._messages.update(response)
                if unstored_messages and settings.DEBUG:
                    raise ValueError('Not all temporary messages could be stored.')
            return response

默认地, message存储对象是django.contrib.messages.fallback.cookie.FallbackStorage, 这个对象实际是同时使用
django.contrib.messages.storage.cookie.CookieStorage和django.contrib.messages.storage.session.SessionStorage.

django.contrib.messages.storage.session.SessionStorage需要SessionMiddleware, 获取message的时候会去request.session中查找session是否存储有message, 存储/获取过程中会使用json来编码解码

.. code-block:: python

    class SessionStorage(BaseStorage):
        """
        Stores messages in the session (that is, django.contrib.sessions).
        """
        session_key = '_messages'

        def __init__(self, request, *args, **kwargs):
            assert hasattr(request, 'session'), "The session-based temporary "\
                "message storage requires session middleware to be installed, "\
                "and come before the message middleware in the "\
                "MIDDLEWARE_CLASSES list."
            super(SessionStorage, self).__init__(request, *args, **kwargs)

        def _get(self, *args, **kwargs):
            """
            Retrieves a list of messages from the request's session.  This storage
            always stores everything it is given, so return True for the
            all_retrieved flag.
            """
            return self.deserialize_messages(self.request.session.get(self.session_key)), True

        def _store(self, messages, response, *args, **kwargs):
            """
            Stores a list of messages to the request's session.
            """
            if messages:
                self.request.session[self.session_key] = self.serialize_messages(messages)
            else:
                self.request.session.pop(self.session_key, None)
            return []

        def serialize_messages(self, messages):
            encoder = MessageEncoder(separators=(',', ':'))
            return encoder.encode(messages)

        def deserialize_messages(self, data):
            if data and isinstance(data, six.string_types):
                return json.loads(data, cls=MessageDecoder)
            return data

django.contrib.messages.storage.cookie.CookieStorage获取message的时候会去request.COOKIES中查找是否有message, 存储/获取过程中会使用settings.SECRET_KEY作为key的salted_hamc编码解码.

.. code-block:: python

    class CookieStorage(BaseStorage):
        """
        Stores messages in a cookie.
        """
        cookie_name = 'messages'
        # uwsgi's default configuration enforces a maximum size of 4kb for all the
        # HTTP headers. In order to leave some room for other cookies and headers,
        # restrict the session cookie to 1/2 of 4kb. See #18781.
        max_cookie_size = 2048
        not_finished = '__messagesnotfinished__'

        def _get(self, *args, **kwargs):
            """
            Retrieves a list of messages from the messages cookie.  If the
            not_finished sentinel value is found at the end of the message list,
            remove it and return a result indicating that not all messages were
            retrieved by this storage.
            """
            data = self.request.COOKIES.get(self.cookie_name)
            messages = self._decode(data)
            all_retrieved = not (messages and messages[-1] == self.not_finished)
            if messages and not all_retrieved:
                # remove the sentinel value
                messages.pop()
            return messages, all_retrieved

        def _update_cookie(self, encoded_data, response):
            """
            Either sets the cookie with the encoded data if there is any data to
            store, or deletes the cookie.
            """
            if encoded_data:
                response.set_cookie(self.cookie_name, encoded_data,
                    domain=settings.SESSION_COOKIE_DOMAIN,
                    secure=settings.SESSION_COOKIE_SECURE or None,
                    httponly=settings.SESSION_COOKIE_HTTPONLY or None)
            else:
                response.delete_cookie(self.cookie_name,
                    domain=settings.SESSION_COOKIE_DOMAIN)

        def _store(self, messages, response, remove_oldest=True, *args, **kwargs):
            """
            Stores the messages to a cookie, returning a list of any messages which
            could not be stored.

            If the encoded data is larger than ``max_cookie_size``, removes
            messages until the data fits (these are the messages which are
            returned), and add the not_finished sentinel value to indicate as much.
            """
            unstored_messages = []
            encoded_data = self._encode(messages)
            if self.max_cookie_size:
                # data is going to be stored eventually by SimpleCookie, which
                # adds its own overhead, which we must account for.
                cookie = SimpleCookie()  # create outside the loop

                def stored_length(val):
                    return len(cookie.value_encode(val)[1])

                while encoded_data and stored_length(encoded_data) > self.max_cookie_size:
                    if remove_oldest:
                        unstored_messages.append(messages.pop(0))
                    else:
                        unstored_messages.insert(0, messages.pop())
                    encoded_data = self._encode(messages + [self.not_finished],
                                                encode_empty=unstored_messages)
            self._update_cookie(encoded_data, response)
            return unstored_messages

        def _hash(self, value):
            """
            Creates an HMAC/SHA1 hash based on the value and the project setting's
            SECRET_KEY, modified to make it unique for the present purpose.
            """
            key_salt = 'django.contrib.messages'
            return salted_hmac(key_salt, value).hexdigest()

        def _encode(self, messages, encode_empty=False):
            """
            Returns an encoded version of the messages list which can be stored as
            plain text.

            Since the data will be retrieved from the client-side, the encoded data
            also contains a hash to ensure that the data was not tampered with.
            """
            if messages or encode_empty:
                encoder = MessageEncoder(separators=(',', ':'))
                value = encoder.encode(messages)
                return '%s$%s' % (self._hash(value), value)

        def _decode(self, data):
            """
            Safely decodes an encoded text stream back into a list of messages.

            If the encoded text stream contained an invalid hash or was in an
            invalid format, ``None`` is returned.
            """
            if not data:
                return None
            bits = data.split('$', 1)
            if len(bits) == 2:
                hash, value = bits
                if constant_time_compare(hash, self._hash(value)):
                    try:
                        # If we get here (and the JSON decode works), everything is
                        # good. In any other case, drop back and return None.
                        return json.loads(value, cls=MessageDecoder)
                    except ValueError:
                        pass
            # Mark the data as used (so it gets removed) since something was wrong
            # with the data.
            self.used = True
            return None

Form wizard
-------------

`表单向导 <https://docs.djangoproject.com/en/1.7/ref/contrib/formtools/form-wizard/>`_, 用户需要填写多个表单, 每个表单必须验证通过才会显示下一个表单, 直到最后一个表单提交通过为止.

简单来说, 流程就是定义多个form, 根据存储的类型, 定义一个form wizard view, 默认有SessionWizardView和CookieWizardView.

CookieWizardView中使用了django.contrib.formtools.wizard.storage.cookie.CookieStorage作为存储对象, 在这个存储对象中会对cookie签名,签名的时候使用settings.SECRET_KEY作为key

.. code-block:: python

    class CookieStorage(storage.BaseStorage):
        encoder = json.JSONEncoder(separators=(',', ':'))

        def __init__(self, *args, **kwargs):
            super(CookieStorage, self).__init__(*args, **kwargs)
            self.data = self.load_data()
            if self.data is None:
                self.init_data()

        def load_data(self):
            try:
                # 这里获取cookie
                data = self.request.get_signed_cookie(self.prefix)
            except KeyError:
                data = None
            except BadSignature:
                raise WizardViewCookieModified('WizardView cookie manipulated')
            if data is None:
                return None
            return json.loads(data, cls=json.JSONDecoder)

        def update_response(self, response):
            super(CookieStorage, self).update_response(response)
            if self.data:
                # 这里update的时候对cookie签名
                response.set_signed_cookie(self.prefix, self.encoder.encode(self.data))
            else:
                response.delete_cookie(self.prefix)

在response.set_signed_cookie中, 有

.. code-block:: python

    from django.core import signing

    def set_signed_cookie(self, key, value, salt='', **kwargs):
        value = signing.get_cookie_signer(salt=key + salt).sign(value)
        return self.set_cookie(key, value, **kwargs)

在django.core.signing.get_cookie_signer中, 有

.. code-block:: python

    def get_cookie_signer(salt='django.core.signing.get_cookie_signer'):
        Signer = import_string(settings.SIGNING_BACKEND)
        # 用settings.SECRET_KEY作为签名的key
        key = force_bytes(settings.SECRET_KEY)
        return Signer(b'django.http.cookies' + key, salt=salt)

Password Reset
----------------

django默认的重设密码会发送一封带有重设密码链接的邮件给用户, 然后用户请求链接的时候, 检查链接上带着的token. 其中token的生成过程是在django.contrib.auth.tokens.PasswordResetTokenGenerator

.. code-block:: python

    class PasswordResetTokenGenerator(object):
        """
        Strategy object used to generate and check tokens for the password
        reset mechanism.
        """
        def make_token(self, user):
            """
            Returns a token that can be used once to do a password reset
            for the given user.
            """
            return self._make_token_with_timestamp(user, self._num_days(self._today()))

        def check_token(self, user, token):
            """
            Check that a password reset token is correct for a given user.
            """
            # Parse the token
            try:
                ts_b36, hash = token.split("-")
            except ValueError:
                return False

            try:
                ts = base36_to_int(ts_b36)
            except ValueError:
                return False

            # Check that the timestamp/uid has not been tampered with
            if not constant_time_compare(self._make_token_with_timestamp(user, ts), token):
                return False

            # Check the timestamp is within limit
            if (self._num_days(self._today()) - ts) > settings.PASSWORD_RESET_TIMEOUT_DAYS:
                return False

            return True

        def _make_token_with_timestamp(self, user, timestamp):
            # timestamp is number of days since 2001-1-1.  Converted to
            # base 36, this gives us a 3 digit string until about 2121
            ts_b36 = int_to_base36(timestamp)

            # By hashing on the internal state of the user and using state
            # that is sure to change (the password salt will change as soon as
            # the password is set, at least for current Django auth, and
            # last_login will also change), we produce a hash that will be
            # invalid as soon as it is used.
            # We limit the hash to 20 chars to keep URL short
            key_salt = "django.contrib.auth.tokens.PasswordResetTokenGenerator"

            # Ensure results are consistent across DB backends
            login_timestamp = user.last_login.replace(microsecond=0, tzinfo=None)

            value = (six.text_type(user.pk) + user.password +
                    six.text_type(login_timestamp) + six.text_type(timestamp))
            hash = salted_hmac(key_salt, value).hexdigest()[::2]
            return "%s-%s" % (ts_b36, hash)

        def _num_days(self, dt):
            return (dt - date(2001, 1, 1)).days

        def _today(self):
            # Used for mocking in tests
            return date.today()

token的生成是在_make_token_with_timestamp方法中, 其中hash值是以(six.text_type(user.pk) + user.password + six.text_type(login_timestamp) + six.text_type(timestamp))来生成的salted_hmac值
, salted_hmac中使用了settings.SECRET_KEY.


若我重设密码的时候, 设置的密码跟原来一致, 并且没有登陆, 则我当天再次请求重设密码的链接的话, 应该还是可以请求的成功的, 但是却显示该链接无效. 则也就是说token改变了.

但是, user的pk, user的last login时间, 当天的时间戳都没变, 只能是password变了, 确切的说是password的hash值变了.

在user.set_password方法中, 有

.. code-block:: python

    from django.contrib.auth.hashers import make_password

    def set_password(self, raw_password):
        self.password = make_password(raw_password)

则在django.contrib.auth.hashers.make_password中, 有

.. code-block:: python

    def make_password(password, salt=None, hasher='default'):
        """
        Turn a plain-text password into a hash for database storage

        Same as encode() but generates a new random salt.
        If password is None then a concatenation of
        UNUSABLE_PASSWORD_PREFIX and a random string will be returned
        which disallows logins. Additional random string reduces chances
        of gaining access to staff or superuser accounts.
        See ticket #20079 for more info.
        """
        if password is None:
            return UNUSABLE_PASSWORD_PREFIX + get_random_string(UNUSABLE_PASSWORD_SUFFIX_LENGTH)
        hasher = get_hasher(hasher)

        if not salt:
            salt = hasher.salt()

        return hasher.encode(password, salt)

密码生成中还一脸一个salt值, 所以, 即使重置的密码不变, 但是密码的hash值一定会变的, 因为salt值变化了.

并且, salt值在密码的hash中, 例如

salt = tfBW269BYR0q

password = pbkdf2_sha256$15000$tfBW269BYR0q$pV+5EbKwRGC2b+/VaeFLA+DpQg0w3w9QioshDFtUea0=



Form Preview
-------------

在django.contrib.formtools.proview.FormPreview中, post之后, 会根据form生成一个hash, 然后比对post进来的hash.

.. code-block:: python

    class FormPreview(object):
        # 省略很多代码
        def preview_post(self, request):
            "Validates the POST data. If valid, displays the preview page. Else, redisplays form."
            f = self.form(request.POST, auto_id=self.get_auto_id())
            context = self.get_context(request, f)
            if f.is_valid():
                self.process_preview(request, f, context)
                context['hash_field'] = self.unused_name('hash')
                context['hash_value'] = self.security_hash(request, f)
                return render_to_response(self.preview_template, context, context_instance=RequestContext(request))
            else:
                return render_to_response(self.form_template, context, context_instance=RequestContext(request))

        def _check_security_hash(self, token, request, form):
            # 比较token和根据form生成的hash
            expected = self.security_hash(request, form)
            return constant_time_compare(token, expected)

        def post_post(self, request):
            "Validates the POST data. If valid, calls done(). Else, redisplays form."
            f = self.form(request.POST, auto_id=self.get_auto_id())
            if f.is_valid():
                if not self._check_security_hash(request.POST.get(self.unused_name('hash'), ''),
                                                 request, f):
                    return self.failed_hash(request)  # Security hash failed.
                return self.done(request, f.cleaned_data)
            else:
                return render_to_response(self.form_template,
                    self.get_context(request, f),
                    context_instance=RequestContext(request))
        # 省略了很多代码
        def security_hash(self, request, form):
            """
            Calculates the security hash for the given HttpRequest and Form instances.

            Subclasses may want to take into account request-specific information,
            such as the IP address.
            """
            return form_hmac(form)

根据form来生成hash的过程是在django.contrib.formtools.utils.form_hmac中, 其中调用salted_hamc.

.. code-block:: python

    def form_hmac(form):
        """
        Calculates a security hash for the given Form instance.
        """
        data = []
        for bf in form:
            # Get the value from the form data. If the form allows empty or hasn't
            # changed then don't call clean() to avoid trigger validation errors.
            if form.empty_permitted and not form.has_changed():
                value = bf.data or ''
            else:
                value = bf.field.clean(bf.data) or ''
            if isinstance(value, six.string_types):
                value = value.strip()
            data.append((bf.name, value))

        pickled = pickle.dumps(data, pickle.HIGHEST_PROTOCOL)
        key_salt = 'django.contrib.formtools'
        return salted_hmac(key_salt, pickled).hexdigest()


Django CSRF
------------

django的csrf校验是在django.contrib.middleware.csrf.Csrfviewmiddleware, csrf的值生成需要settings.SECRET_KEY, 但是校验就不需要了.

核心的过程就是, 比对post过来的名为csrfviewmiddleware的值与cookie中名为csrftoken的值, 一致就是通过, 不一致就不通过.

.. code-block:: python

    class CsrfViewMiddleware(object):

        def process_view(self, request, callback, callback_args, callback_kwargs):
            # 这里省略了代码
            try:
                csrf_token = _sanitize_token(
                    request.COOKIES[settings.CSRF_COOKIE_NAME])
                # Use same token next time
                request.META['CSRF_COOKIE'] = csrf_token
            except KeyError:
                # pass的位置上省掉了很多代码
                pass
            # 继续省略很多代码
            request_csrf_token = ""
            if request.method == "POST":
                request_csrf_token = request.POST.get('csrfmiddlewaretoken', '')

            if request_csrf_token == "":
                # Fall back to X-CSRFToken, to make things easier for AJAX,
                # and possible for PUT/DELETE.
                # AJAX中的csrf是设置/获取http头X-CSRFToken
                request_csrf_token = request.META.get('HTTP_X_CSRFTOKEN', '')

            if not constant_time_compare(request_csrf_token, csrf_token):
                return self._reject(request, REASON_BAD_TOKEN)
            # 最后省略很多代码

所以, 只需要设置post过来的名为csrfviewmiddleware的值与cookie中名为csrftoken的值一致, 随意是什么, 只要相等就行, 就可以通过csrf认证.

post的csrfviewmiddleware容易设置, 而设置同一个域名下的cookie也比较容易, 但是设置跨域的cookie比较麻烦.


另外, AJAX也会出现csrf问题, 在同域名的情况下, 可以直接使用django通过的 `js文件 <https://docs.djangoproject.com/en/1.7/ref/contrib/csrf/>`_ 来解决, 其实也是将要
psot到后台的csrfmiddlewaretoken设置到一个自定义HTTP头X-CSRFToken的头上, 然后django在post的数据拿不到之后, 回去自定义头获取.

