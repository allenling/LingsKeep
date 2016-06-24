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

例如

<my-element query="{{route.query}}" ></my-element>

<my-element name="{{route.params.name}}" ></my-element>

当页面跳转的时候

tmp = {}

tmp['query'] = app.query

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

# line: 546
function onclick(e) {
  // 省略代码
  # line:594
  if (hashbang) path = path.replace('#!', ''); 

# line:379
function Context(path, state) {
  // 省略代码
  # line: 385
  if (hashbang) path = path.replace('#!', ''); 

比如一个链接路径为rootUrl/#!/path/to/somewhere

删除#!之后, 为rootUrl//#!/path/to/somewhere(若rootUrl=/, 则得到//path/to/somewhere)

最简单的方法就是删除掉最前面的/


# line: 546
function onclick(e) {
  // 省略代码
  # line:594
  if (hashbang) {
    path = path.replace('#!', '');
    if (path[1] == '/' && path[0] == path[1]) {
      path = path.substring(1);
    }
  }

# line:379
function Context(path, state) {
  // 省略代码
  # line: 385
  if (hashbang) {
    this.path = this.path.replace('#!', '') || '/';
    if (this.path[1] == '/' && this.path[0] == this.path[1]) {
      this.path = this.path.substring(1);
    }
  }


observer和observers
--------------------------

在element中赋值属性的时候, 会调用observers

<my-element property-one="{{a}}" property-two="{{b}}"></my-element>


obervers: ["test(propertyOne, proertyTwon)"]

在my-element初始化的时候, 会调用test, 并且propertyOne, proertyTwon的值分别为父元素的a和b. 之后一旦属性有修改, 都会调用test, 类似与observer.


若需要监听很多属性, 可以使用observer, 但是这样在初始化的时候, 会调用每一个observer, 例如上面的例子, 会分别调用propertyOne和propertyTwo的observer, 若propertyOne和propertyTwo是共同影响方法的, 则使用observers.

例如, my-element主要是发送api获取数据. 

目标api传参中, page, page_size用于分页, extras用于获取额外的信息, query可以为空, 这样可以返回所有的记录. extras可以不传入, page, page_size必须传入.

我们希望元素可以这样

<my-element page="{{page}}" page-size="{{pageSize}}" extras="{{extras}}" query="{{query}}"></my-element>

其中, query表示url中的querystring. 

显然, 发送api的时候, 受到page, pageSize, extras, query的影响, 所以不能只是对每一个属性单独添加observer了. 可以使用observers.

由于observers中的属性, 必须定义初始值, 所以, 一开始, 可以这么定义

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

但是这样有个问题, api会发送两次, 因为一开始my-element初始化的时候, 会调用observers方法pullRequest. 其中的参数值都为属性的初始化值, 这个时候, query为{}, 则不管父元素的传入的参数, 直接发送了一次api请求.

而在父元素中, 若传入的page, pageSize, extras, query有任一一个不同, 就又会发送第二次api请求. 明显, 这样是不合理的. 我们希望参数由父元素决定.

我们可以明显区分query为null和{}所表达的意思, null表示没有querystring, 是不合理的, 而{}表示querystring为空, 是合理的. 这样, 我们在query的初始值设置为null, 在pullRequest中

判断, 只有query!=null的时候才发送请求, 则上面第一次请求就被过滤掉, 不会发送了


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

