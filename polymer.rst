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


-----------------------------------------------------

vaadin-grid
-------------

要想复制row和cell中的内容,要设置user-select的css为text,默认vaadin-grid中user-select的值为none.

  vaadin-grid {
    -moz-user-select: text;
    -webkit-user-select: text;
    -ms-user-select: text;
  }



1. polymer中,属性的顺序是重要的
2. api的_i_get支持传入queryset
3. changelist元素有两种情况, a. sku list b. 某个package的skus,在情况b中,会有两个请求,第一个是一开始获取所有sku,第二个是搜索具体某个pacakage的请求,能不能避免第一个请求?
   由于在sku-list-element中observers了page和pageSize, 则在package-sku-element中不传入page给sku-element,有sku-list-element自己去设置page, 根本原因是page到底是在list元素里面控制还是交给paginator来控制.
4. 倾向与list元素只是接受query dict,然后generateReuqest,parseResponse, 而组织page,page_size等参数的行为在父元素中.
5. 解决方式是用中间变量来限制数据绑定


<sku-list-element page="{{_page}}" page-size="{{_pageSize}}"></sku-list-element>

page: {
  type: Number,
  value: 1
},
pageSize: {
  type: Number,
  value: 100
},
_page: {
  type: Number,
},
_pageSize: {
  type: Number,
},

observers: ['pageAndSize(page, pageSize)]

pageAndSize: function (page, pageSize) {
  if (this.package) {
    this._page = this.page;
    this._pageSize = this.pageSize;
  }
}

6. <element name="status"></element>, <element name="{{name}}"></element>, <element name$="{{status}}"></element>的区别
7. 设置vaadin-grid的renderer的时候, 比如要显示状态的名称, 记得rerender一下vaadin-grid
   
   <collocation-status-filter-element on-items-changed="reRenderGrid"></collocation-status-filter-element>

   var statusRenderer = function (cell) {
     cell.element.innerHTML = cell.data; 
     if (cell.data == null) {
       return;
     }
     let items = self.$.statusFilter.items;
     let index = self.$.statusFilter.$.collocationStatusFilter.$.vaadin._indexOfValue(String(cell.data));
     if (index < 0) {
       return;
     }
     cell.element.innerHTML = items[index][1];
   }

   reRenderGrid: function (e) {
     if (!this.$.collocationsTable || !e.detail.value || e.detail.value.length == 0) {
       return;
     }
     this.$.collocationsTable.refreshItems();
   }
8. 显示detail的时候, 外键还是用extras直接获取比较好
  
  也可以使用foreign-key-object-element来显示

  <shopowner-object-element pk="{{item.pk}}" on-value-changed="showShopowner"></shopowner-object-element>

  <paper-item >{{shopownerName}}</paper-item>

  showShopowner: function (e) {
    ...
    this.ShopownerName = e.detail.value['name'];
  }


  而像状态之类的只能获取list的属性, 只能两次渲染才能保证一定显示出来.因为不确定获取status的ajax是在获取item(如搭配)的ajax结束之前或者之后结束的

  <collocation-status-list-element on-value-changed="showStatus"></collocation-status-list-element>

  <paper-item >{{statusName}}</paper-item>

  item: {
    type: Object,
    observer: '_itemChange'
  }
  _itemChange: function () {
    this.showStatus();
  }
  showStatus: function () {
    for () {
    }
  }



6. psk中package changelist 会发送两次package list的请求, 在polymer element中不存在.
------------------------------------------------------------------------------------
   原因是query和page,pageSize传参问题. 在page-changelist-element中, query使用了observer,同时observers监听了page,pageSize,所以第一次初始化的时候, package-changelist-element中的page和pageSize
   是默认值,则触发observers, 请求第一次,之后query从psk中传入{}(page.js中,若没有querystring,则data.query就是{}), 触发query的observer(query不给value,就是undefined). 若希望设置query的默认值为{},这样
   psk传入{}的时候不触发query的observer这也不行,因为设置了value了必然会触发observer的.
   
   1. 解决方法就是设置一个中间值_query,package-lst-element的filterParams绑定到_query, 若query有oldValue == undefined && Object.keys(newValue).length == 0的时候,不赋值_query,否则_query=query.
      但是若一开始query就有值,如一开始query={"shopowner": 1}的时候,依然会发送两个请求,第一个是不带querystring的请求, 第二个就是带有querystring的请求.所以, 我们希望page, pageSize和query一起考虑,而不是
      分别对他们进行监听. 
   2. 可以使用observers[pullRequest(page, pageSize, filterParams)], 这个时候polymer会将会在这三个参数都赋值完成之后触发observers方法, 关键是三个参数都赋值之后才触发该observers方法. 所以,针对1中最后还是
      会发起两个请求在这里就只会一个带有querystring的请求了.
