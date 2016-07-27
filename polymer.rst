Polymer
=========

version: 1.0


vaadin combox box
------------------

传入的items的itemNamePath和itemValuePath一定要是字符串
items={'name': '1', 'value': 'v1'}
否则选中会有问题


dom-if
-------

使用dom-if进行if...else渲染的时候,注意的是有可能this.$会选中不到元素

需要用Polymer.dom(this.root).querySelector()来选择元素

若dom-if为false, 则不会删除元素,只是设置元素的style为display: none.

changed事件
-------------

polymer中的绑定对象的时候, 要注意绑定方法, 否则会失效.
    
最好是将对象先保存到一个中间变量, 如tmp, 然后直接赋值, 这样调用了polymer的set方法, 才会
检测到绑定对象的update.

let tmp = {}
for (let c in bindData) {
  tmp[c] = bindData[c];
}
this.data = bindData;
(等效于this.set('data', bindData))

23.2 或者直接绑定对象的属性, 但这样不灵活.

<element data-name="{{data.name}}"></element>


文档:
Binding to structured data

For a path binding to update, the path value must be updated in one of the following ways:

1. Using a {{site.project_title}} property binding to another element. 

2. Using the set API, which provides the required notification to elements with registered interest.


所以, 一般一旦调用set, 则都会触发change事件的, 因为就算键名和键值都一样, 两个对象使用等于操作(==, ===, Object.is等等)都不一定是相等的.

若监听某个属性的变化, 则必须调用属性的set方法, site.project_title = value.


vaadin-grid
-------------

要想复制row和cell中的内容,要设置user-select的css为text,默认vaadin-grid中user-select的值为none.

  vaadin-grid {
    -moz-user-select: text;
    -webkit-user-select: text;
    -ms-user-select: text;
  }

page.js的一些bug
------------------

当hash=true, 并且直接访问地址为rootUrl/#!/path/to/somewhere会404, 这是因为在page.js中, 删除路径中的#!的时候, 有bug

// line: 546
function onclick(e) {
  // 省略代码
  // line:594
  if (hashbang) path = path.replace('#!', ''); 

// line:379
function Context(path, state) {
  // 省略代码
  // line: 385
  if (hashbang) path = path.replace('#!', ''); 

比如一个链接路径为rootUrl/#!/path/to/somewhere

删除#!之后, 为rootUrl//#!/path/to/somewhere(若rootUrl=/, 则得到//path/to/somewhere)

最简单的方法就是删除掉最前面的/


// line: 546
function onclick(e) {
  // 省略代码
  // line:594
  if (hashbang) {
    path = path.replace('#!', '');
    if (path[1] == '/' && path[0] == path[1]) {
      path = path.substring(1);
    }
  }

// line:379
function Context(path, state) {
  // 省略代码
  // line: 385
  if (hashbang) {
    this.path = this.path.replace('#!', '') || '/';
    if (this.path[1] == '/' && this.path[0] == this.path[1]) {
      this.path = this.path.substring(1);
    }
  }


observer和observers
--------------------------

调用顺序是先observer, 再observers.

observers和observer的区别是

1. observers监听多个属性变动, 并且在初始化的时候只调用一次. 而observer会分别调用多个属性的observer.

2. 顺序上, 一定是先observer, 再到observers.

3. observers中, 必须是所有的参数都部位undefined, 才会调用, 所以polymer建议observers的每一个property都给一个value.

4. 经过初始化之后, observers中任意一个property变化, 都会触发observers, 这个时候observers的行为就像observer一样.

例子 

my-element主要是发送api获取数据. 

我们希望元素可以这样

<my-element page="{{page}}" page-size="{{pageSize}}" extras="{{extras}}" query="{{query}}"></my-element>

其中, query表示url中的querystring. 

显然, 发送api的时候, 受到page, pageSize, extras, query的影响, 所以不能只是对每一个属性单独添加observer了. 可以使用observers.

由于observers中的属性, 必须定义初始值, 所以, 一开始, 可以这么定义

.. code-block::

    page: {
      type: String,
      value: 1
    }
    
    pageSize: {
      type: String,
      value: 100
    }
    
    extras: {
      type: String,
      value: ''
    }
    
    query: {
      type: Object,
      value: {}
    }
    
    observers: ["pullRequest(page, pageSize, extras, query)"]
    
    pullRequest: function(page, pageSize, extras, query) {
      // 发送请求
    }

这样, 父元素中可以这么使用:

<my-element page="{{page}}" page-size="{{pageSize}}" extras="{{extras}}" query="{{query}}"></my-element>

但是这样有个问题, api会发送两次, 因为一开始my-element初始化的时候, 会调用observers方法pullRequest. 其中的参数值都为属性的初始化值, 直接发送了一次api请求.

而在父元素中, 若传入的page, pageSize, extras, query有任一一个不同, 比如pageSize=20, 则触发observers, 会发送第二次api请求. 明显, 这样是不合理的. 我们希望参数由父元素决定.

我们可以明显区分query为null和{}所表达的意思, null表示没有querystring, 是不合理的, 而{}表示querystring为空, 是合理的. 这样, 我们在query的初始值设置为null, 在pullRequest中

判断, 只有query!=null的时候才发送请求, 则上面第一次请求就被过滤掉, 不会发送了

.. code-block::

    page: {
      type: String,
      value: 1
    }
    
    pageSize: {
      type: String,
      value: 100
    }
    
    extras: {
      type: String,
      value: ''
    }
    
    query: {
      type: Object,
      value: null
    }
    
    observers: ["pullRequest(page, pageSize, extras, query)"]
    
    pullRequest: function(page, pageSize, extras, query) {
      if (query == null) {
        return;  
      }
      // 发送请求
    }

Polymer property 绑定顺序
---------------------------

Polymer中property(polymer对象定义的properties)绑定的顺序是element上attribute(dom上定义的attribute)的倒序.

1. 例子
~~~~~~~~

.. code-block::

    // my-element的定义
    Polymer({
      is: 'my-element',
      properties: {
        query: {
          type: Object,
          observer: 'queryChange',
          value: null
        },
        pkg: {
          type: String,
          observer: 'pkgChange',
          value: null
        }
      },
      observers: ['pullRequest(query, pkg)'],
      pkgChange: function (newP, oldP) {
        console.log(newP);
        console.log(oldP);
        console.log('log pkg');
      },
      queryChange: function (newQ, oldQ) {
        console.log(newQ);
        console.log(oldQ);
        console.log('log query');
      }
      pullRequest: function (query, pkg) {
        
      }
    });

    // 在psk的routing.html中

    function setAppInfo(data) {
      let tmp = {};
      tmp['query'] = data.query;
      tmp['params'] = {};
      app.set(app.route, tmp);
      // 调用Polymer.set方法去强制触发change event
      // Polymer对象还提供了很多这种操作, 必须使用Polymer内置的方法才能触发数据更新
      // 比如要更新dom-repeat中的items中的某个元素的key, 必须调用Polymer.set(), 直接array[0].key = newValue的话, dom-repeat并不会重现渲染items.
      for (let key in data.params) {
        //  这样是不行的 app[app.route]['params'][key] = data.params[key];
        app.set(app.route + '.params.' + key, data.params[key]);
      }
    }

    page('/packages', function(data) {
      app.route = 'myRoute';
      app.appName = 'myElement';
      setAppInfo(data);
      setFocus(app.route);
    });

在psk的index.html使用my-element

1. <my-element pkg="{{myRoute.params.pkg}}" query="{{myRoute.query}}" ></my-element>
   1.1 在app.set的时候, 先触发queryChange, 之后再触发pullRequest(由query change触发的), 这时因为pkg不为undefined 并且query为空字典, 发送第一次请求.
   1.2 触发pkgChange(由params change触发), 这个时候new pkg = undefined, 则并不会触发pullRequest
   1.3 之后循环set params, 触发pkg change, 这时候pkg不为undefined, 接着触发pullRequest, 发送第二次请求

2. <my-element query="{{query}}" pkg="{{params.pkg}}" ></my-element>
   2.1 app.set, 触发pkgChange(由params change触发), 这个时候new pkg = undefined, 则并不会触发pullRequest
   2.2 接着触发queryChange, 之后再触发pullRequest(由query change触发的), 这时因为pkg=undefined, 不触发pullRequest.
   2.3 之后循环set params, 触发pkg change, 这时候pkg不为undefined, 接着触发pullRequest, 发送一次请求

2. 绑定过程
~~~~~~~~~~~~

<my-element data-one="{{dataOne}}" data-two="{{dataTwo}}" ></my-element>

这里, 会先触发data-two的observer, 之后是跟data-two有关的observers, 之后是data-one的observer, 之后是跟data-one有关的observers.

这是因为polymer在建立绑定的时候, 就是根据attribute的倒序来绑定的, 也就是先绑定data-two, 再绑定data-one

.. code-block::

    // polymer.html:179-228
    _parseNodeAttributeAnnotations: function (node, annotation) {
      // 这里获取element的attributes
      var attrs = Array.prototype.slice.call(node.attributes);
      //这里倒序去绑定
      for (var i = attrs.length - 1, a; a = attrs[i]; i--) {
        var n = a.name;
        var v = a.value;
        var b;
        if (n.slice(0, 3) === 'on-') {
        node.removeAttribute(n);
        annotation.events.push({
          name: n.slice(3),
          value: v
        });
        } else if (b = this._parseNodeAttributeAnnotation(node, n, v)) {
          annotation.bindings.push(b);
        } else if (n === 'id') {
          annotation.id = v;
        }
      }
    },
    // 这里是绑定的过程
    _parseNodeAttributeAnnotation: function (node, name, value) {
      var parts = this._parseBindings(value);
      if (parts) {
        var origName = name;
        var kind = 'property';
        if (name[name.length - 1] == '$') {
          name = name.slice(0, -1);
          kind = 'attribute';
        }
        var literal = this._literalFromParts(parts);
        if (literal && kind == 'attribute') {
          node.setAttribute(name, literal);
        }
        if (node.localName === 'input' && origName === 'value') {
          node.setAttribute(origName, '');
        }
        node.removeAttribute(origName);
        var propertyName = Polymer.CaseMap.dashToCamelCase(name);
        if (kind === 'property') {
          name = propertyName;
        }
        return {
          kind: kind,
          name: name,
          propertyName: propertyName,
          parts: parts,
          literal: literal,
          isCompound: parts.length !== 1
        };
      }
    },

dom-repeat update Array
--------------------------


dom-repeat绑定的Array, 更新的时候, 必须调用polymer自己的set方法才能使得dom-repeat刷新Array.

<template is="dom-repeat" items="{{myArray}}" >
  <p >{{item.name}}</p>
</template>

myMethod: function () {
  // can not update dom-repeat Array
  myArray[0].name = 'new name';

  // must invoke this.set
  this.set('myArray.0.name', 'new name'); 
}

Polymer提供了一系列array操作方法来帮助更新dom-repeat中的Array.
文档 https://www.polymer-project.org/1.0/docs/devguide/properties#array-mutation
