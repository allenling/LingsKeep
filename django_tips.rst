DJANGO
=======

以下基于>=1.8,<1.9版本

事务
====

django的事务控制在wsgi中

django.core.handlers.wsgi.py(#158)

.. code-block:: python

    class WSGIHandler(base.BaseHandler):
        def __call__(self, environ, start_response):
            response = self.get_response(request)

django.core.handlers.base(#132)

.. code-block:: python

    def get_response(self, request):
        # 这里去wrap事务
        wrapped_callback = self.make_view_atomic(callback)
        try:
            response = wrapped_callback(request, *callback_args, **callback_kwargs)
        except Exception as e:
            # 省略代码
            pass


ORM和测试
===========================

Django的orm只需要配置好settings中的DATABASES, PTHONPATH中包含有models就行.

若是一个project只是使用orm, 并且app都是其他project的app的时候, INSTALLED_APPS可以为空, 这并不影响orm的使用. 但是django的test就必须配置INSTALLED_APPS, 因为测试
一般都是新建数据库以及新建表来测试, 这个时候会调用migration, 而migration会根据migration的文件来绝对是migrate还是新建表, 所以一般在settings文件中将app的migration的目录配置到
一个没有使用, 也可以是不存在的目录, 然后migration在每次测试的时候就都是新建数据库新建表了. 

.. code-block:: python

    MIGRATIONS = {'app1': 'never_used_path'}

并且测试的时候最好使用sqlite, 内存型数据库, 测试完之后, 释放内存, 不存留sqlite文件. 其他的数据库就必须配置用户权限等等, 比较麻烦.

**上面说的, orm不需要INSTALLED_APPS是错误的, 最好必须有INSTALLED_APPS. 例子:** 

Query: SELECT "djangoapp_person"."id", "djangoapp_person"."name" FROM 

 "djangoapp_person" INNER JOIN "djangoapp_group_members" ON ( "djangoapp_person"."id" = 

 "djangoapp_group_members"."person_id" ) WHERE "djangoapp_group_members"."group_id" = 1

不在INSTALLED_APPS中加载ManyToMany所指向的app的话, 报错, 则是models的opts初始化问题

.. code-block:: python

    # django.db.models.options.get_field_by_name(line413):
    def get_field_by_name(self, name):
        try:
            try:
                return self._name_map[name]
            except AttributeError:
                cache = self.init_name_map()
                return cache[name]
        except KeyError:
            raise FieldDoesNotExist('%s has no field named %r'
                    % (self.object_name, name))

如上, 寻找m2m的时候, 主要是看opts中存的fields有没有. 而最终, init_name_map中会负责找出所有的ManyToManyField.

INSTALLED_APPS中不写带有ManyToManyField的models(包括在model中定义的和reverse的), 则_name_map中始终找不到 ManyToManyField. INSTALLED_APPS中的app只是在setup的时候导入models module而已, 貌似没什么特别的.

django.setup()中只是简单的import_module.import_module(app.models)模块而已, 若我在其他脚本中先导入了ManyToManyField的类, 效果应该和setup中导入models mudle一样的~~!?


若不在model中显式指定app_label或者把model所在的app写在INSTALLED_APPS, 会报这个warnings, 1.9之后就报错了.

unicode: Model class __main__.Person doesn't declare an explicit app_label and either isn't in an 
 application in INSTALLED_APPS or else was imported before its application was loaded. This will 
 no longer be supported in Django 1.9.


源码证明问题跟import没关系, 是django寻找ManyToManyfield关系的机制问题

model的fields都存在model._meta(opts).fields中, 而不定义在model中的ManyToManyfield字段会寻找所有的app.models
来搜索对应的ManyToManyfield.

.. code-block:: python

    # 源码在db.models.options(line577)
    def _fill_related_many_to_many_cache(self):
        cache = OrderedDict()
        parent_list = self.get_parent_list()
        for parent in self.parents:
            for obj, model in parent._meta.get_all_related_m2m_objects_with_model():
                if obj.field.creation_counter < 0 and obj.model not in parent_list:
                    continue
                if not model:
                    cache[obj] = parent
                else:
                    cache[obj] = model
        # 这里会遍历所有的app的models来寻找ManyToManyfield
        for klass in self.apps.get_models():
            if not klass._meta.swapped:
                for f in klass._meta.local_many_to_many:
                    if (f.rel
                            and not isinstance(f.rel.to, six.string_types)
                            and self == f.rel.to._meta):
                        cache[f.related] = None

所以若带有ManyToManyfield的model的app并没有设置在INSTALLED_APPS或者为在models中显式指定app的话, 是找不到
ManyToManyField的.

所以, 一句话, django1.7之后的Application会带上很多配置信息, 所以INSTALLED_APPS是非常必须的.


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

django upload
=============================

上传文件的时候, 文件应该使用rb模式打开. python打开文件的默认的text模式(r)会将文件的换行符转换成系统指定的, windows下就是\n\r, 而linux下的就是\n. 若客户端是windows, 打开文件使用
text模式, 上传到linux之后, 很可能造成文件失效. 比如windows下客户端上传一个xlxs文件到linux服务器, 则文件内容会被xlrd判别为无效文件, 若客户端使用读取二进制(rb)的模式读取文件, 则不会
出现这个问题. linux下打开文件的text模式和二进制模式是相等的.

文档: https://docs.python.org/2/library/functions.html#open

django model form
==================

clean中调用self.instance
-------------------------
django model form中, 若field有默认值 ,
则self.instance初始化的时候会自动赋上默认值(null=True, blank=True相当于默认值为None), 若field没有默认值, 则初始化的时候则self.instance并不会有这个属性

比如field: package
若没有默认值, 在clean中调用self.instance.package会报没有这个属性错误

在changeform_view方法中, 调用form.save的时候, commit是false, 真正save是在save_model方法中, 所有要自定义外键等对象赋值, 就必须在save_model中操作

.. code-block:: python

    # chageform_view方法中save_form总是commit=False
    def save_form(self, request, form, change):
        return form.save(commit=False)

    # 由于commit=False, 则form.save中并不会save_m2m
    def save(self, commit=True):
    if self.instance.pk is None:
        fail_message = 'created'
    else:
        fail_message = 'changed'
    # save_instance函数负责save model
    return save_instance(self, self.instance, self._meta.fields,
                         fail_message, commit, self._meta.exclude,
                         construct=False)
    # save_instance函数中判断commit是否是True来绝对是否save_m2m
    def save_instance(form, instance, fields=None, fail_message='saved',
                  commit=True, exclude=None, construct=True):
        # 省略了很多代码
        if commit:
            instance.save()
            save_m2m()
        else:
            form.save_m2m = save_m2m
        return instance

    # 真正的保存到数据库是在changeform_view的save_model方法
    def save_model(self, request, obj, form, change):
        obj.save()

model_form的field的初始化和显示
--------------------------------

若使用django admin, 则change_form中初始化多少个field并不是model_form决定的, 而是ModelAdmin中的fieldsets决定的, fieldsets形式为

.. code-block:: python

    fieldsets = (
        (None, {
            'fields': ('url', 'title', 'content', 'sites')
        }), )

而显示多少个field则是changeform_view方法中初始化的adminForm来决定的

.. code-block:: python

    def changeform_view(...):

        adminForm = helpers.AdminForm(
            form,
            list(self.get_fieldsets(request, obj)),
            self.get_prepopulated_fields(request, obj),
            self.get_readonly_fields(request, obj),
            model_admin=self)

这里get_readonly_field返回的字段在html上就是一串文本, 也就是只读了.

而在model_form中的__init__中也可以设置某个field是只读的

.. code-block:: python

    class MyModelForm(ModelForm):
        def __init__(...):
            super(MyModelForm, self).__init__(...)
            # 设置field只读
            self.fields['field'].readonly = True

**但是这里的只读只是不可输入, 但是组件还是为显示出来, 比如还是一个input框, 但是鼠标不可输入而已.**

django bulk_create
========================

方法1 normal bulk create:

.. code-block:: python

    obj=Obj.save()

    obj.save()

    OtherObj.objects.bulk_create([OtherObj(obj=obj, name=name), ...])

sql语句为:

INSERT INTO `myapp_otherobj` (`obj_id`, `name`) VALUES (2593, 50001), ...

方法2 lazy bulk create:

.. code-block:: python

    obj = Obj()

    tmp_bulk.append(lambda: obj.pk)

    tmp_bulk = [lambda: OtherObj.objects.bulk_create([OtherObj(obj=obj, name=name), ...])]

    obj.save()

    for _ in tmp_bulk:
        _()

sql语句为:

INSERT INTO `myapp_otherobj` (`obj_id`, `name`) VALUES (NULL, 50001), ...

**很明显, lazy执行bulk_create的时候, obj的pk并没有设置上, 但是实际上, lazy bulk_create之前已经调用obj.save了. 并且打印出来的obj的pk是存在的, Why?**

猜测应该跟lazy没关系, 而是外键的obj.save()前后的问题.

*形式1*

obj.save()

OtherObj.objects.bulk_create([OtherObj(obj=obj, ...), ...])

*形式2*

tmp = [OtherObj(obj=obj, ...), ...]

obj.save()

OtherObj.objects.bulk_create(tmp)

形式1是可以的

形式2是不可以的(lazy的方式也是这种, 先组装好list, 在最后调用bulk_create)

区别就是上面显示的insert语句的区别. 组建sql的时候, 代码如下

.. code-block:: python

    # django.db.models.sql.compiler(860)

    def as_sql(self):
        # 省略了代码
        # 这里当f是外键的时候, 走到f.pre_save(obj, True)中
        if has_fields:
            params = values = [
                [
                    f.get_db_prep_save(getattr(obj, f.attname) if self.query.raw else f.pre_save(obj, True), connection=self.connection)
                    for f in fields
                ]
                for obj in self.query.objs
            ]

    # 会调用在django.db.models.fields.__init__(597)中的pre_save方法

    def pre_save(self, model_instance, add):
        # 这里attname就是外键在数据库里面的字段名, 这里是obj_id
        return getattr(model_instance, self.attname)

所以, 组建sql的时候, 会去找外键在model中数据库的字段的属性值, 这里就是otherobj.obj_id, 形式1中会在生成mode实例的时候赋值上obj_id, 而形式2则不会. 两者应该都是找外键obj的pk
但是为何不一样.?

区别在model.__init__方法中

.. code-block:: python

    # django.db.models.base(435)

    def __init__(self, *args, **kwargs):
        # 省略了代码
        # 若传进来的参数有外键, 则赋值外键
        if is_related_object:
            setattr(self, field.name, rel_obj)
        else:
            setattr(self, field.attname, val)

    #  setattr最后会调用django.db.models.fields.related(583)中的__set__方法, 设置model中外键的pk
    def __set__(self, instance, value):
        # 省略了代码
        # 这里instance是model实例, lh_field.attrname则是外键的数据库字段名, 如obj, 这里attrname就是obj_id, value就是外键对象, 即obj实例, rh_field.attname则是关联外键的数据库字段, 这里是id
        try:
            setattr(instance, lh_field.attname, getattr(value, rh_field.attname))
        except AttributeError:
            setattr(instance, lh_field.attname, None)


所以, model.__init__方法一开始就将外键的数据库字段名属性给赋值好了, 若是形式1的情况, 自然是能将外键的pk赋值上, 若是形式2, 则赋值为None, 组建sql的时候也就是为None.

所以也就是, bulk_create必须在外键obj.save之后, 也就是外键obj必须有pk

proxy model permission
========================

创建proxy model的时候, model的类名必须跟原来的类名不一致才能创建出权限.
**若proxy model和source model不再同一个app下, 则为proxy model创建的permissions指向的content type不是proxy model所在的app, 而是source model所在的app.!**


可以使用shell来更新proxy model的permission

.. code-block:: python

    In [9]: for app in apps:
        contenttypes = ContentType.objects.filter(app_label=app)
        permissions = Permission.objects.filter(codename__contains='_%s'%app)
        for p in permissions:
            for c in contenttypes:
                if c.name in p.name:
                    print p, p.content_type_id, c, c.pk
                    p.content_type=c
                    p.save()

源码:

django.core.management.commands.migrate(165)
---------------------------------------------

.. code-block:: python

    emit_post_migrate_signal(created_models, self.verbosity, self.interactive, connection.alias)


django.core.management.sql(256)
---------------------------------------------

.. code-block:: python

    def emit_post_migrate_signal(created_models, verbosity, interactive, db):
        # Emit the post_migrate signal for every application.
        for app_config in apps.get_app_configs():
            if app_config.models_module is None:
                continue
            if verbosity >= 2:
                print("Running post-migrate handlers for application %s" % app_config.label)
            models.signals.post_migrate.send(
                sender=app_config,
                app_config=app_config,
                verbosity=verbosity,
                interactive=interactive,
                using=db)
            # For backwards-compatibility -- remove in Django 1.9.
            models.signals.post_syncdb.send(
                sender=app_config.models_module,
                app=app_config.models_module,
                created_models=created_models,
                verbosity=verbosity,
                interactive=interactive,
                db=db)

完成migration之后, 会发送post_migrate, reciver中包含了create permissions
-----------------------------------------------------------------------------

.. code-block:: python

    def send(self, sender, **named):
        responses = []
        if not self.receivers or self.sender_receivers_cache.get(sender) is NO_RECEIVERS:
            return responses
        # reciver中有create permissions
        for receiver in self._live_receivers(sender):
            response = receiver(signal=self, sender=sender, **named)
            responses.append((receiver, response))
        return responses

create permissions在这里: django.contrib.auth.management(62)
------------------------------------------------------------------

.. code-block:: python

    def create_permissions(app_config, verbosity=2, interactive=True, using=DEFAULT_DB_ALIAS, **kwargs):
        # 上面省略了代码
        searched_perms = list()
        ctypes = set()

        # 这个for去寻找app中所有的model的permission
        for klass in app_config.get_models():
            # 注意, 若proxy model的类名和被proxy的model的类型一致, 这里会得到被proxy的model的ContentType, 这样proxy model的权限就不会被创建
            # 若proxy model和source model不再同一个app下, 这里get_for_model拿到的meta永远是source model的meta, 也就是拿到的content type是source model的content type!
            ctype = ContentType.objects.db_manager(using).get_for_model(klass)
            ctypes.add(ctype)
            for perm in _get_all_permissions(klass._meta, ctype):
                searched_perms.append((ctype, perm))

        # 这里去数据库查找该app下所有的permissions
        all_perms = set(Permission.objects.using(using).filter(
            content_type__in=ctypes,
        ).values_list(
            "content_type", "codename"
        ))

        # 两者相差, 找出需要创建的permissions
        perms = [
            Permission(codename=codename, name=name, content_type=ct)
            for ct, (codename, name) in searched_perms
            if (ct.pk, codename) not in all_perms
        ]
        # 最后省略了代码

获取content type的过程: django.contrib.contenttypes.models_module
--------------------------------------------------------------------

.. code-block:: python

    # 这里总是返回source model的meta, 而不会是proxy model的meta
    def _get_opts(self, model, for_concrete_model):
        if for_concrete_model:
            model = model._meta.concrete_model
        elif model._deferred:
            model = model._meta.proxy_for_model
        return model._meta

    def get_for_model(self, model, for_concrete_model=True):
        opts = self._get_opts(model, for_concrete_model)
        # 省略代码


大文件下载
-----------

首先, open(filename)会返回一个file-like对象, 这个对象是自己的一个迭代器, 也就是说open一个文件并没有加载文件所有的内容

file.readline, file.readlines(重复调用readline)是一行一行返回, file.read才是读取所有的内容到内存.

1. 静态文件应该交由服务器来处理

2. 可以在django中配置使用服务器来发送文件

.. code-block:: python

    from django.utils.encoding import smart_str

    response = HttpResponse(mimetype='application/force-download')
    response['Content-Disposition'] = 'attachment; filename=%s' % smart_str(file_name)
    response['X-Sendfile'] = smart_str(path_to_file)
    # It's usually a good idea to set the 'Content-Length' header too.
    # You can also set any other required headers: Cache-Control, etc.
    return response

这里的X-Sendfile标识位表示使用服务器来发送文件, 前提是服务器已经开启了mod_xsendfile模块.

**但是有个问题是mod_xsendfile不支持带有unicode字符的文件名**

3. 可用用StreamingHttpResponse, 流式下载

django文档的建议, 就是将一个迭代器传入Streaminghttpresponse,返回这个httpresponse.

.. code-block:: python

    import csv

    from django.utils.six.moves import range
    from django.http import StreamingHttpResponse

    class Echo(object):
        def write(self, value):
            """Write the value by returning it, instead of storing in a buffer."""
            return value

    def some_streaming_csv_view(request):
        """A view that streams a large CSV file."""
        rows = (["Row {}".format(idx), str(idx)] for idx in range(65536))
        pseudo_buffer = Echo()
        writer = csv.writer(pseudo_buffer)
        response = StreamingHttpResponse((writer.writerow(row) for row in rows),
                                         content_type="text/csv")
        response['Content-Disposition'] = 'attachment; filename="somefilename.csv"'
        return response


其中QuerySet是可以当做迭代器来使用

django.db.models.query

.. code-block:: python

    class QuerySet(object):
        def __iter__(self):
            """
            The queryset iterator protocol uses three nested iterators in the
            default case:
                1. sql.compiler:execute_sql()
                   - Returns 100 rows at time (constants.GET_ITERATOR_CHUNK_SIZE)
                     using cursor.fetchmany(). This part is responsible for
                     doing some column masking, and returning the rows in chunks.
                2. sql/compiler.results_iter()
                   - Returns one row at time. At this point the rows are still just
                     tuples. In some cases the return values are converted to
                     Python values at this location.
                3. self.iterator()
                   - Responsible for turning the rows into model objects.
            """
            self._fetch_all()
            return iter(self._result_cache)

        def iterator(self):
            # 省略代码
            pass


django.setup如何加载model
--------------------------

.. code-block:: python

    # 在django.setup()中

    for app_config in self.app_configs.values():
        all_models = self.all_models[app_config.label]
        # 这里导入models
        app_config.import_models(all_models)

找到models.py, 这里model的module名字是写死在代码中的

.. code-block:: python

    # 在django.apps.config.py中
    # models的module名称是写死的
    MODELS_MODULE_NAME = 'models'
    def import_models(self, all_models):
        # Dictionary of models for this app, primarily maintained in the
        # 'all_models' attribute of the Apps this AppConfig is attached to.
        # Injected as a parameter because it gets populated when models are
        # imported, which might happen before populate() imports models.
        self.models = all_models

        if module_has_submodule(self.module, MODELS_MODULE_NAME):
            # 这里导入models
            models_module_name = '%s.%s' % (self.name, MODELS_MODULE_NAME)
            self.models_module = import_module(models_module_name)


对field进行复杂聚合
--------------------

比如, django搜索某个月之内的数据, 思路基本上是设置一个Function(即一个mysql的expression), 提取处月份, 然后annotate. django中的COUNT之类的也是这样一个思路, 提取除数据然后annotate.

或者, 自定义一个field, 叫month_field

auto_now_add/auto_now
----------------------

loaddata的时候,即使datetime设置为auto_add_now, fixture中datetime也必须给值, 因为auto_now_add/auto_now是在save的时候, django自己去设置的. 而loaddata

不管是lodadata还是objects.create/save, 都是调用到django.db.models.base.Model.save_base, 区别在下面:

.. code-block:: python

   # django.db.models.sql.compiler

   class SQLInsertCompiler(SQLCompiler):
   
       def as_sql(self):
           
           if has_fields:
               params = values = [
               [f.get_db_prep_save(
                    getattr(obj, f.attname) if self.query.raw else f.pre_save(obj, True),
                    connection=self.connection
                ) for f in fields
               ]
               for obj in self.query.objs
              ]

其中, lodadata的时候, self.query.raw为True, 而getattr(obj, f.attname)=None, 但是DateField/DatetimeField是null=False, blank=Ture的, 并不允许为None, 所以报错.
而objects.create/save的时候, self.query.raw=False, 则调用DateField/DatetimeField中的pre_save, 在pre_save中, 会对auto_now_add/auto_now判断, 将当前日期/时间赋值给object.

而在lodadata中, 会调用django.core.serializers.deserialize获得一个django.core.serializers.base.Deserializer对象, 调用Deserializer.save()方法, 在save方法中, 手动设置了sql.raw=True

.. code-block:: python

   class DeserializedObject(object):
       def save(self, save_m2m=True, using=None):
           # 这里设置了raw=True
           models.Model.save_base(self.object, using=using, raw=True)
           if self.m2m_data and save_m2m:
               for accessor_name, object_list in self.m2m_data.items():
                   setattr(self.object, accessor_name, object_list)

           # prevent a second (possibly accidental) call to save() from saving
           # the m2m data twice.
           self.m2m_data = None

而关于raw参数, 在save_base的注释中有说明. raw表示在save之前, 不会对model的所有field有改动.

.. code-block:: python

  def save_base(self, raw=False, force_insert=False,
                force_update=False, using=None, update_fields=None):
      """
      Handles the parts of saving which should be done only once per save,
      yet need to be done in raw saves, too. This includes some sanity
      checks and signal sending.
  
      The 'raw' argument is telling save_base not to save any parent
      models and not to do any changes to the values before save. This
      is used by fixture loading.
      """
      # 省略代码
      pass


将foreign key的控件换成datalist
---------------------------------

最直接的想法是在model定义的时候更换他的widget, 但是foreign key的控件不能通过__init__方法覆盖, 而是有一个hook:

django.db.models.fields.related: 1754

.. code-block:: python

    def formfield(self, **kwargs):
        db = kwargs.pop('using', None)
        if isinstance(self.rel.to, six.string_types):
            raise ValueError("Cannot create form field for %r yet, because "
                             "its related model %r has not been loaded yet" %
                             (self.name, self.rel.to))
        defaults = {
            'form_class': forms.ModelChoiceField,
            'queryset': self.rel.to._default_manager.using(db),
            'to_field_name': self.rel.field_name,
        }
        defaults.update(kwargs)
        return super(ForeignKey, self).formfield(**defaults)


admin 中自定义foreignkey和manytomany的queryset和widget

admn中有两个方法:

formfield_for_manytomany和formfield_for_foreignkey

kwargs中传入widget和queryset

如在group admin中, 获取manytomany的permissions时, kwargs是这样的

.. code-block:: python

  class GroupAdmin(admin.ModelAdmin):
      search_fields = ('name',)
      ordering = ('name',)
      filter_horizontal = ('permissions',)
  
      def formfield_for_manytomany(self, db_field, request=None, **kwargs):
          if db_field.name == 'permissions':
              qs = kwargs.get('queryset', db_field.rel.to.objects)
              # Avoid a major performance hit resolving permission names which
              # triggers a content_type load:
              kwargs['queryset'] = qs.select_related('content_type')
          return super(GroupAdmin, self).formfield_for_manytomany(
              db_field, request=request, **kwargs)


ForeignKey在form控件是forms.Modelchoicefield, 可以重载, 方法是在ForeignKey中的formfield方法中(django.db.models.fields.related:1754)

.. code-block:: python

    def formfield(self, **kwargs):
        db = kwargs.pop('using', None)
        if isinstance(self.rel.to, six.string_types):
            raise ValueError("Cannot create form field for %r yet, because "
                             "its related model %r has not been loaded yet" %
                             (self.name, self.rel.to))
        defaults = {
            'form_class': forms.ModelChoiceField,
            'queryset': self.rel.to._default_manager.using(db),
            'to_field_name': self.rel.field_name,
        }
        defaults.update(kwargs)
        return super(ForeignKey, self).formfield(**defaults)

form中的changed_data是form来判断某个field是否有修改的方法, 主要是对比initial_data和post过来的data做对比(django.forms.forms:408)

.. code-block:: python

    def changed_data(self):
        if self._changed_data is None:
            self._changed_data = []
            # XXX: For now we're asking the individual widgets whether or not the
            # data has changed. It would probably be more efficient to hash the
            # initial data, store it in a hidden field, and compare a hash of the
            # submitted data, but we'd need a way to easily get the string value
            # for a given field. Right now, that logic is embedded in the render
            # method of each widget.
            for name, field in self.fields.items():
                prefixed_name = self.add_prefix(name)
                data_value = field.widget.value_from_datadict(self.data, self.files, prefixed_name)
                if not field.show_hidden_initial:
                    initial_value = self.initial.get(name, field.initial)
                    if callable(initial_value):
                        initial_value = initial_value()
                else:
                    initial_prefixed_name = self.add_initial_prefix(name)
                    hidden_widget = field.hidden_widget()
                    try:
                        initial_value = field.to_python(hidden_widget.value_from_datadict(
                            self.data, self.files, initial_prefixed_name))
                    except ValidationError:
                        # Always assume data has changed if validation fails.
                        self._changed_data.append(name)
                        continue
                if hasattr(field.widget, '_has_changed'):
                    warnings.warn("The _has_changed method on widgets is deprecated,"
                        " define it at field level instead.",
                        RemovedInDjango18Warning, stacklevel=2)
                    if field.widget._has_changed(initial_value, data_value):
                        self._changed_data.append(name)
                elif field._has_changed(initial_value, data_value):
                    self._changed_data.append(name)
        return self._changed_data

