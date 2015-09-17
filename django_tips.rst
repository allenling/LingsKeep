ORM和测试
===========================

Django的orm只需要配置好settings中的DATABASES, PTHONPATH中包含有models就行.

若是一个project只是使用orm, 并且app都是其他project的app的时候, INSTALLED_APPS可以为空, 这并不影响orm的使用. 但是django的test就必须配置INSTALLED_APPS, 因为测试
一般都是新建数据库以及新建表来测试, 这个时候会调用migration, 而migration会根据migration的文件来绝对是migrate还是新建表, 所以一般在settings文件中将app的migration的目录配置到
一个没有使用, 也可以是不存在的目录, 然后migration在每次测试的时候就都是新建数据库新建表了. 

.. code-block:: python

    MIGRATIONS = {'app1': 'never_used_path'}

并且测试的时候最好使用sqlite, 内存型数据库, 测试完之后, 释放内存, 不存留sqlite文件. 其他的数据库就必须配置用户权限等等, 比较麻烦.

Settings
==============

Settings是一个lazy-object, 当调用django.setup的时候, 会配置logging, 这个时候, settings就evaluate了.

.. code-block:: python

    def setup():
        from django.apps import apps
        from django.conf import settings
        from django.utils.log import configure_logging

        # lazy的settings会在这里evaluate
        configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
        apps.populate(settings.INSTALLED_APPS)

根据文档可知, 一般建议是把settings都写在文件中, 不建议settings.x=y的形式.

Django中的application是一个独立于project的, 在django.setup的时候, 会populate各个INSTALLED_APP, 只是加载module和加载application的conf而已, application的conf
是针对各个application配置. 而有一个第三方的django app, 叫django-appconf, 也是为各个app配置自定义的settings, 只是使用场景不太一样.

从使用方式上区别比较明显, django自己的app conf不一赖settings

.. code-block:: python

    In [1]: from django.apps import apps

    In [2]: apps.get_app_config('djangoapp')
    Out[2]: <DjangotestConfig: djangoapp>

而在django-appconf中settings, 是将对应app的conf中的配置导入settings对象中(是在settings文件的基础上, 添加配置到settings对象中),
以django-avatar这个app为例子(django-avatar依赖django-appconf)

.. code-block:: python

    In [3]: from avatar.conf import settings

    In [4]: settings.AVA
    settings.AVATAR_ALLOWED_FILE_EXTS     settings.AVATAR_DEFAULT_URL           settings.AVATAR_HASH_USERDIRNAMES     settings.AVATAR_STORAGE_DIR
    settings.AVATAR_AUTO_GENERATE_SIZES   settings.AVATAR_GRAVATAR_BACKUP       settings.AVATAR_MAX_AVATARS_PER_USER  settings.AVATAR_THUMB_FORMAT
    settings.AVATAR_CACHE_TIMEOUT         settings.AVATAR_GRAVATAR_BASE_URL     settings.AVATAR_MAX_SIZE              settings.AVATAR_THUMB_QUALITY
    settings.AVATAR_CLEANUP_DELETED       settings.AVATAR_GRAVATAR_DEFAULT      settings.AVATAR_RESIZE_METHOD
    settings.AVATAR_DEFAULT_SIZE          settings.AVATAR_HASH_FILENAMES        settings.AVATAR_STORAGE

而django-appconf的具体实现就是直接settattr(settings, var). 使用django-appconf, 声明一个继承于appconf.AppConf的类, 类中定义具体的配置, 配置也可以是方法的形式.以avatar为例子.

.. code-block:: python

    # avatar/conf.py

    class AvatarConf(AppConf):
        DEFAULT_SIZE = 80
        # 省略了很多属性, 下面的方法也可以来配置settings
        def configure_auto_generate_avatar_sizes(self, value):
            return value or getattr(settings, 'AVATAR_AUTO_GENERATE_SIZES',
                                    (self.DEFAULT_SIZE,))

定义好自己的配置类之后, 只要加载了这个module, 也就是avatar.conf, 就会自动把自定义的settings配置到settings中.这是因为在AppConf中比较hack的实现.

.. code-block:: python

    # appconf/base.py
    class AppConf(six.with_metaclass(AppConfMetaClass)):
        # 省略了很多代码
        pass

而在AppConf的metaclass, 也就是AppConfMetaClass中, __new__方法会加载django.conf.settings, 并且将自定义的属性赋值到settings中.

.. code-block:: python

    class AppConfOptions(object):

        def __init__(self, meta, prefix=None):
            self.prefix = prefix
            # 这里导入django.conf.settings
            self.holder_path = getattr(meta, 'holder', 'django.conf.settings')
            self.holder = import_attribute(self.holder_path)
            self.proxy = getattr(meta, 'proxy', False)
            self.required = getattr(meta, 'required', [])
            self.configured_data = {}

    class AppConfMetaClass(type):

        def __new__(cls, name, bases, attrs):
            # 省略了很多代码...

            # 生成一个实例
            new_class = super_new(cls, name, bases, {'__module__': module})

            # 又省略了很多代码...
            # 这里添加类AppConfOptions, 并且实例化, 此时就加载了django.conf.settings
            new_class.add_to_class('_meta', AppConfOptions(meta, prefix))
            # 依然省略了代码...
            new_class._configure()
            for name, value in six.iteritems(new_class._meta.configured_data):
                prefixed_name = new_class._meta.prefixed_name(name)
                # 这里setattr将自定义变量配置到django.conf.settings中
                setattr(new_class._meta.holder, prefixed_name, value)
                new_class.add_to_class(name, value)

        def _configure(cls):
            # the ad-hoc settings class instance used to configure each value
            obj = cls()
            # 这里就将配置自定义的变量, 添加app前缀
            for name, prefixed_name in six.iteritems(obj._meta.names):
                default_value = obj._meta.defaults.get(prefixed_name)
                value = getattr(obj._meta.holder, prefixed_name, default_value)
                callback = getattr(obj, "configure_%s" % name.lower(), None)
                if callable(callback):
                    value = callback(value)
                cls._meta.configured_data[name] = value
            cls._meta.configured_data = obj.configure()

__new__和six.with_metaclass
===============================

__new__方法是一个实例生成的时候调用的静态方法(print type.__new__的时候, 输出是function), __init__是初始化一个实例, 并且只有当__new__返回一个实例, 才会调用__init__方法.
而six.with_metaclass用来兼容python2和python3中的metaclass语法的, 特别是一个类由一个metaclass和继承于一个base class的时候.

.. code-block:: python

    from six import with_metaclass


    class Meta(type):
        def __new__(cls, *args, **kwargs):
            print 'in Meta __new__'
            return type.__new__(cls, *args, **kwargs)


    class Base(object):
        def __new__(cls, *args, **kwargs):
            print 'in Base __new__'
            return object.__new__(cls, *args, **kwargs)

    class MyClass(with_metaclass(Meta, Base)):
        pass

当第一次导入MyClass的时候会生成Meta类的类, 也就是可以看成创建了一个Meta类的实例.

.. code-block:: python

    In [1]: from test import MyClass
    in Meta __new__

    In [2]: x=MyClass()
    in Base __new__

__new__参考: http://agiliq.com/blog/2012/06/__new__-python/

