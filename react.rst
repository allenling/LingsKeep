react v15.6.1

**不用url hash(#!)算什么spa, 每次跳页面还要请求服务器获取html算什么spa**

react and polymer
====================

react and web component
https://reactjs.org/docs/web-components.html

virtual dom and shadow dom
https://stackoverflow.com/questions/36012239/is-shadow-dom-fast-like-virtual-dom-in-react-js

在react中, 页面刷新是先destroy之前的node, 比如<div id="root" ><Node1 ></Node1></div>, re-render之后, 先删除node1然后渲染node2, <div id="root" ><Node2 ></Node2></div>.

而polymer只是把Node1给设置为display: hidden

如果hidden的话，事件还会被监听到的，比如Node1监听了scroll, 被hidden之后，依然还是会监听到scroll, 同样的react也会，react只是删掉dom而已, 对象还在，所以react也会出现这样的问题


root element
===============

react只会在id=root里面渲染



immutable(again!)
===================

React elements are immutable. Once you create an element, you can’t change its children or attributes. An element is like a single frame in a movie: it represents the UI at a certain point in time.
With our knowledge so far, the only way to update the UI is to create a new element, and pass it to ReactDOM.render().

https://reactjs.org/docs/rendering-elements.html


react element是immutable的, 更新element的唯一方法就是创建一个新的元素然后render, 因为react只会在id=root的element里面渲染，所以新的
element会替换掉原来的元素

之所以说immutable, again!是因为看看elasticsearch, paxos, 都是immutable的~~~~


virtual dom
============

更新元素只能重新rerender, 这样性能不行呀，所以virtual dom的作用就是比对新的element和之前的element, 只更新必要的部分~~在react中, 只会调用render一次啦


jsx
====

一种js和html混写的语法, 进步或倒退(毕竟js, css, html拆分是这么多年来的方向), 还是挺好?

componment
===========

componment是一个函数, 返回html代码的, 可以是类, 但是继承自react的componment(带有render方法), 或者是函数，返回html代码


props and state
================

Whether you declare a component as a function or a class, it must never modify its own props

State is similar to props, but it is private and fully controlled by the component.

use state correctly
---------------------

Do Not Modify State Directly
++++++++++++++++++++++++++++++++

this.State.a = 1这样并不能更新compoment, 应该是this.SetState({'a': 1}), 这样像不像polymer的set.

State Updates May Be Asynchronous
+++++++++++++++++++++++++++++++++++++++

React may batch multiple setState() calls into a single update for performance.
react会把多个setState的操作合起来一起执行的

Because this.props and this.state may be updated asynchronously, you should not rely on their values for calculating the next state.
就像上面说的, 不能依赖state里面的value

State Updates are Merged
+++++++++++++++++++++++++

没太看懂


data flow(数据绑定,数据流)
==============================

例子:

// 一个组件
function FormattedDate(props) {
  return <h2>It is {props.date.toLocaleTimeString()}.</h2>;
}
//传值
//这里的this是当前element而不是指FormattedDate
<FormattedDate date={this.state.date} />


数据绑定是父->子, 上->下, 单向的不是双向的(双工的)


ref
====

如果要执行子element的方法的话，可以使用ref.

ref是element对应的实例， 比如<Child ref={(child) => {this.cd = child}}, 这里ref接收一个函数，函数参数就是Child这个element的实例，然后我们把this.cd赋值于child, 这样this.cd就是child实例了

也可以暴露子element的dom给parent, 也还是通过ref的形式

function CustomTextInput(props) {
  return (
    <div>
      // 这里input会被ref到prop.inputRef, 这样父element就可以拿到了
      <input ref={props.inputRef} />
    </div>
  );
}

class Parent extends React.Component {
  render() {
    return (
      <CustomTextInput
        // 父element将接收子element的inputRef这个prop
        inputRef={el => this.inputElement = el}
      />
    );
  }
}

https://reactjs.org/docs/refs-and-the-dom.html



如果子element要改变父element的state的话, 可以这样https://ourcodeworld.com/articles/read/409/how-to-update-parent-state-from-child-component-in-react

就是将一个函数fn, fn的作用是修改父element的state, function fn () {this.setState({'a': 1});}, 通过prop传入到子element中，然后在子element中调用this.prop.fn


event
==========

驼峰命名去定义事件, 比如onClick, 后面接函数名，不能像html那样用字符串

Another difference is that you cannot return false to prevent default behavior in React. You must call preventDefault explicitly.

阻止事件冒泡不能返回false, 必须显示调用preventDefault

**定义event handler需要绑定this**

class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // This binding is necessary to make `this` work in the callback
    // 这里的bind可以类比与python中函数传入self, 成为跟实例绑定的方法
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(prevState => ({
      isToggleOn: !prevState.isToggleOn
    }));
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.state.isToggleOn ? 'ON' : 'OFF'}
      </button>
    );
  }
}

ReactDOM.render(
  <Toggle />,
  document.getElementById('root')
);

有个js的知识点:  In JavaScript, class methods are not bound by default. If you forget to bind this.handleClick and pass it to onClick, this will be undefined when the function is actually called.

是不是很傻，我在类里面定义方法居然不是默认就绑定到实例的.

关于js的bind: https://www.smashingmagazine.com/2014/01/understanding-javascript-function-prototype-bind/

箭头函数定义法
---------------

class LoggingButton extends React.Component {
  handleClick() {
    console.log('this is:', this);
  }

  render() {
    // This syntax ensures `this` is bound within handleClick
    return (
      <button onClick={(e) => this.handleClick(e)}>
        Click me
      </button>
    );
  }
}

但是不太好，有re-render的性能问题:

The problem with this syntax is that a different callback is created each time the LoggingButton renders. In most cases, this is fine. However, if this callback is passed as a prop to lower components, those components might do an extra re-rendering. We generally recommend binding in the constructor or using the class fields syntax, to avoid this sort of performance problem.


if...else render
==================

如果return null, 则组件不会显示


react ssr
==========

其实ssr没什么的, 就是django以前渲染html那一套嘛, 只不过是用node然后调用react组件来在node里面渲染react而已

比较坑的是react的机制

基本流程
----------


写react之后, 打成一个node包(也就是编译成es5而已, babel一下了), 然后写一个server.js, 里面有调用renderToString方法

然后再express的view(这个就是django里面的view)里面require一下这个server.js, 然后传入prop(初始化的map)调用.

.. code-block:: 

    function ssr(url, prop) {

        var mp = (<ExperienceDetail state={props} />);
        let markup = renderToString(mp);

        return markup
    } 




window is undefined
---------------------------------------

ssr中, Node环境是没有window的，所以必须注意去掉window或者加上type window == undefined这样的判断

特别是ComponentWillMount和ComponentDidMount这两个方法要注意



渲染了两次
---------------


ssr的时候图方便，用了<StaticRouter>

.. code-block:: 

    let markup = renderToString(
      <StaticRouter location={url} context={context}>
        <div>
                  <Route exact path="/" component={App} />
                  <Route path="/Login" component={Login} />
                  <Route path='/experience-list' component={List}></Route>
                  <Route path='/My' component={My}></Route>
                  <Route path='/Editor' component={Editor}></Route>
                  <Route path='/experience-detail' render={()=><ExperienceDetail state={props} />}></Route>
                  <Route path='/preview' component={PreView}></Route>
                  <Route path='/PublishSuccess' component={PublishSuccess}></Route>
                  <Route path='/userrole' component={UserRole}></Route>
                  <Route path='/about' component={About}></Route>
                  <Route path='/m-detail' render={()=><MDetail state={props} />}></Route>
                  <Route path='/m-my' component={MMY} ></Route>
                  <Route path='/home' component={MHOME}></Route>          
    
    
                  <Route exact path="/" component={App} />
    
                  <Route path="/Login" component={Login} />
                  <Route path='/experience-list' component={List}></Route>
                  <Route path='/My' component={My}></Route>
                  <Route path='/Editor' component={Editor}></Route>
                  <Route path='/experience-detail' render={()=><ExperienceDetail state={props} />}></Route>
                  <Route path='/preview' component={PreView}></Route>
                  <Route path='/PublishSuccess' component={PublishSuccess}></Route>
                  <Route path='/userrole' component={UserRole}></Route>
                  <Route path='/about' component={About}></Route>
    
       </div>
    
      </StaticRouter>

但是发现ssr会渲染两次，只要把StaticRouter的形式换成if else 就好了, 下面是被babel转过的，但是if else的意思是一样的


.. code-block:: 

  //server.js

  var ssr = function (url, props) {
    if (url.startsWith('/experience-detail')) {
      var mp = (<div ><ExperienceDetail state={props} /></div>);
    }else if (url.startsWith('/m-detail')) {
      var mp = (<div ><MDetail state={props} /></div>);
    }else if (url == '/') {
      var mp = (<div > <App /></div>);
    }else if (url.startsWith('/home')) {
      var mp = (<div ><MHOME /></div>);
    }
    let markup = renderToString(mp);
    return markup;
  }
  
  module.exports = ssr;



**这里注意的是需要在组件之前加一个div**

比如var mp = (<div ><ExperienceDetail state={props} /></div>);这里的div就是需要额外加的


渲染速度
----------

一般性能都损失在react渲染上(别说api, api不会很慢的，相信我), 一般的解决办法只能加缓存咯~~~

client是否会重新渲染
---------------------

react再ssr返回的文件里面会带有一个checksum的的属性

<div data-reactroot="" data-reactid="1" data-react-checksum="46739197">

通过chrome的sources来看能看到，然后再elements里面看(就是检查元素)是看不到的，被react删除掉了

checksum的作用就是react自己对比ssr过来的dom和在client的dom是否是一致的，不一致就重新渲染，一致就更新其他的dom

至于怎么通过checksum说明ssr出来的dom和client自己渲染的一致，不知道，有人说不报错就是对的，那好吧.

可以这样, 校验checksum的代码是在react-dom/lib/ReactMarkupChecksum.js:41:  canReuseMarkup: function (markup, element), 可以加一个console打印一下，

**之前网上的人说checksum被删除就是可以复用了, 但是相反, checksum被删除说明react没有复用dom, 被网上的答案骗了**

下面是追寻答案的过程:

  1. 通过chrome的performance(注意看screenshot)显示确实dom被重建了，估计是因为用户信息栏的区别~~~因为现在ssr的是没办法去获取当前用户的信息,
     要获取的话只能通过cookie传入用户的api token, 当然，更好的方法是把顶部用户信息和文章给拆成两个组件(这种本来就应该拆分的，被前端写到一起了,
     所以兼职前端写得真的很烂).
  
  2. 校验checksum的代码是在react-dom/lib/ReactMarkupChecksum.js:43(http://www.crmarsh.com/react-ssr/)

     .. code-block:: 
        // console.log是我加的打印信息

        canReuseMarkup: function (markup, element) {
          var existingChecksum = element.getAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
          existingChecksum = existingChecksum && parseInt(existingChecksum, 10);
          var markupChecksum = adler32(markup);
          var can_resue = markupChecksum === existingChecksum;
          console.log('------------client side markup: ');
          try {
            if (element.length == undefined) {
              console.log('checksum class element: ' + element.className);
            }else {
              console.log('checksum class element: ' + element[0].className);
            }   
          }   
          catch (e) {
            console.log('get checksum class element error: ' + e); 
            console.log(element);
          }   
          console.log('------------existingChecksum: ' + existingChecksum +'----markupChecksum: ' + markupChecksum);
          console.log(markup);
          console.log('------------client side can resue react: ' + can_resue);
          return can_resue;
        }

     删除checksum的代码在node_modules/react-dom/lib/ReactMount.js:471
     
     .. code-block:: 

        _mountImageIntoNode: function (markup, container, instance, shouldReuseMarkup, transaction) {
          !isValidContainer(container) ? process.env.NODE_ENV !== 'production' ? invariant(false, 'mountComponentIntoNode(...): Target container is not valid.') : _prodInvariant('41') : void 0;

          if (shouldReuseMarkup) {
            var rootElement = getReactRootElementInContainer(container);
            if (ReactMarkupChecksum.canReuseMarkup(markup, rootElement)) {
              ReactDOMComponentTree.precacheNode(instance, rootElement);
              return;
            } else {
              var checksum = rootElement.getAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
              # 这里不能复用dom的话会删除checksum!!!!!!!!!!!!!
              rootElement.removeAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
     

  3. 根本原因，react的ComponentWillMount发起异步获取任务之后，不会等待数据加载完成，就继续render了,所以，client side第一次render出来永远是一个"空"的dom!

    3.1 可以在3.2的里面打印出信息得出是"空"的dom, react方法调用顺序:
        constructor()
        componentWillMount()
        render()
        componentDidMount()

    3.2 There’s a “gotcha,” though: An asynchronous call to fetch data will not return before the render happens. This means the component will render with empty data at least once.
        https://daveceddia.com/where-fetch-data-componentwillmount-vs-componentdidmount/

    3.3 估计只能在compoint里面先fetch data在render~~~但是这样在单页面应用的时候效果就不好，因为这样也没会卡住，这跟异步页面应用相违背了
        或者把ssr出来的数据放到html里面，比如在html里面window.__ssr__state__ = data(估计只能);

    3.4 但是这样这样的前提还是要把当前登录用户的信息拆分出来，不要放到一个component里面，因为ssr请求的时候不会带上user的token，无法拿到当前登录用户的数据，然后虽然也可以
        在cookie上带上token，但是这样~~~还是先不带吧
  
    3.5 判断是否需要复用dom在react的判断是：　var shouldReuseMarkup = containerHasReactMarkup && !prevComponent && !containerHasNonRootReactChild;
        其中containerHasReactMarkup就是<div id="root"></div>, ssr渲染之后其中必然有tag，所以client side就会去检查

    3.6 给出state都一样，初始化详情页的时候, client的markup和server side的markup老是不一样, 看了下, 在编辑器初始化的时候
        <div className='editorContainer'>
            <Editable
                placeholder="分享你的经验 ..."
                defaultValue={this.state.initialContent}
                readOnly={true}
                ref={node => { this.editor = node }}
            />
        </div>
        其中ref可能没赋值对, 跟ref没关系, 这个ref接收一个回调函数，回调函数的参数是组件的实例(这里就是Editable的实例，然后箭头函数的意思就是this.editor赋值为编辑器)                

    3.7 看来大家都是把state发送到client(https://github.com/facebook/react/issues/9681第一个回复), 可以在index.html里面<body ><script>window.__initState__ = {state}</script>...</body>
  
    3.8 出现一个现象: 第一次ssr出来到客户端的是，是可以复用的，之后就不可以了，原因是使用的编辑器slate会增加一个data-key的属性,
  　　　并且ssr的时候data-key会增加, 而client side是不会增加了, 比如
  　　　第一次的时候<editor data-key=1></editor>, 之后<editor data-key=x></editor>, 这个x值会按固定步长增加，比如7, 13, 19,...
        官方说增加了一个resetKeyGenerator函数(https://github.com/ianstormtaylor/slate/issues/53) 可以在ssr的时候key的值每次都是0



  


